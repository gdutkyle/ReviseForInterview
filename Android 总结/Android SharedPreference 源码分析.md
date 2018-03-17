# Android SharedPreferences 源码分析 #

## 初识SharedPreferences   
我们可以简单的认为SharedPreferences（一下简称sp）是Android系统提供给我们的一个轻量级的存储系统。我们可以使用sp的存储一些轻量的信息，比如姓名、账号这些。通过简单的代码，就把我们需要传入的数据，保存到我们的sp中。比如如下代码：  

    	SharedPreferences sp=this.getSharedPreferences("TEST",Context.MODE_PRIVATE);
        SharedPreferences.Editor editor=sp.edit();
        editor.putString("key","values");
        editor.putInt("key1",1);
        sp.edit().commit();
        
        //get from sp when we need
        String value= sp.getString("key","");
        Log.i("========",value);  

## SharePreferences的初始化  
通过上面的SharedPreferences sp=this.getSharedPreferences("TEST",Context.MODE_PRIVATE);我们可以知道SharePreference 是在contextImpl中进行初始化的。那么我们去到ContextImpl中去查看  

     @Override
    public SharedPreferences getSharedPreferences(String name, int mode) {
        // At least one application in the world actually passes in a null
        // name.  This happened to work because when we generated the file name
        // we would stringify it to "null.xml".  Nice.
        if (mPackageInfo.getApplicationInfo().targetSdkVersion <
                Build.VERSION_CODES.KITKAT) {
            if (name == null) {
                name = "null";
            }
        }

        File file;
        synchronized (ContextImpl.class) {
            if (mSharedPrefsPaths == null) {
                mSharedPrefsPaths = new ArrayMap<>();
            }
            file = mSharedPrefsPaths.get(name);
            if (file == null) {
                file = getSharedPreferencesPath(name);
                mSharedPrefsPaths.put(name, file);
            }
        }
        return getSharedPreferences(file, mode);
    }

从这段代码我们一步一步分析  

**1 通过synchronized字段我们可以看到，sp的初始化是线程安全的**  

**2 android首先会通过一个map<String,file>去存储当前安卓系统生成的sp文件。我们在前面传入的TEST name，就会在这个时候进行map的key查找，如果找不到，那么会选择去创建一个sp文件：file = getSharedPreferencesPath(name);**
  
好了，我们就下来去看android系统是如何去创建一个sp的：  

    @Override
    public File getSharedPreferencesPath(String name) {
        return makeFilename(getPreferencesDir(), name + ".xml");
    }
    private File getPreferencesDir() {
        synchronized (mSync) {
            if (mPreferencesDir == null) {
                mPreferencesDir = new File(getDataDir(), "shared_prefs");
            }
            return ensurePrivateDirExists(mPreferencesDir);
        }
    }

好了，这也就是我们平时root手机后，去我们包名下面shared_prefs文件夹下找到的后缀为.xml文件。  

## SharePrefenences 的 Editor  
根据我们上面的Editor使用方法，我们可以大致的认为，Editor就是我们sp的一个编辑器，我们通过Editor去为sp赋值并且commit（apply），我们就下来就去看看Editor的源码  

    public interface Editor {
     Editor putString(String key, @Nullable String value);
     Editor putStringSet(String key, @Nullable Set<String> values);
     Editor putInt(String key, int value);
     ......
    }

其实Editor就是一个接口，我们可以使用editor去修改或者增加一个值，所有的editor操作都是支持批量操作的，并且在你调用commit或者apply之前，都不会将本次Editor提交给sp。  
Editor的实现类EditorImp是在SharePreferencesImpl类中定义的，我们重点看editor的commit()和apply()方法  

     public boolean commit() {
            MemoryCommitResult mcr = commitToMemory();
            SharedPreferencesImpl.this.enqueueDiskWrite(
                mcr, null /* sync write on this thread okay */);
            try {
                mcr.writtenToDiskLatch.await();
            } catch (InterruptedException e) {
                return false;
            }
            notifyListeners(mcr);
            return mcr.writeToDiskResult;
        }

从代码中我们可以分析，首先我们执行commit的时候，先把commit的内容提交到memory中，然后我们在把这个内存中的数据写入到disk(硬盘)中。核心方法是，我们在SharePreferencesImp中的writeToFile(MemoryCommitResult mcr)方法，在这个过程中，android还做了一次系统备份。  
接下来我们来看apply()方法  

    public void apply() {
            final MemoryCommitResult mcr = commitToMemory();
            final Runnable awaitCommit = new Runnable() {
                    public void run() {
                        try {
                            mcr.writtenToDiskLatch.await();
                        } catch (InterruptedException ignored) {
                        }
                    }
                };

            QueuedWork.add(awaitCommit);

            Runnable postWriteRunnable = new Runnable() {
                    public void run() {
                        awaitCommit.run();
                        QueuedWork.remove(awaitCommit);
                    }
                };

            SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);

            // Okay to notify the listeners before it's hit disk
            // because the listeners should always get the same
            // SharedPreferences instance back, which has the
            // changes reflected in memory.
            notifyListeners(mcr);
        }  

apply()方法和commit()方法最大的差别就是commit是在UI线程(当前线程)中操作的，而apply是另外启动一个线程去操作的。与commit一样，当有两个editor对同一个sp进行操作操作的时候，只会以当前最后一个生效的操作为准。

## SharePreference的putXXXX()和getXXXX()操作   
我们简单的以**putString**(String key, @Nullable String value)和**getString**(String key, @Nullable String defValue)方法为例子进行分析  

 	private final Map<String, Object> mModified = Maps.newHashMap();

    public Editor putString(String key, @Nullable String value) {
            synchronized (this) {
                mModified.put(key, value);
                return this;
            }
        }  

    @Nullable
    public String getString(String key, @Nullable String defValue) {
    	synchronized (this) {
    			awaitLoadedLocked();
    			String v = (String)mMap.get(key);
   				return v != null ? v : defValue;
    		}
    }

由代码可以看到，put操作其实就是把当前的key和value put进一个hashMap里面，而get方法就是从把put进去hashmap中的值给get出来，代码比较简单，不做具体分析  

## SharePreference 的变动通知：registerOnSharedPreferenceChangeListener  

     public void registerOnSharedPreferenceChangeListener(OnSharedPreferenceChangeListener listener) {
    	synchronized(this) {
   		mListeners.put(listener, mContent);
    	}
    }

这个方法主要用来监听sp的变化的，我用的比较少，大家可以根据自己的业务进行使用。 
 
## 其它  
其实SP也是支持跨进程调用的，只要我们在初始化的时候，传入 SharedPreferences sp=mActivity.getSharedPreferences("TEST", Context.MODE_MULTI_PROCESS);但是这个方法已经被官方弃用了。官方建议如果需要的话，使用**contentProvider**代替使用，关于cp的原理分析，我下一篇可以简单进行分析