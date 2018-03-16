Android SharedPreference 源码分析

初识SharePreference  
我们可以简单的认为SharePreference（一下简称sp）是Android系统提供给我们的一个轻量级的存储系统。我们可以通过sp简单的存储一些轻量的信息，比如姓名、账号这些。我们可以通过简单的代码，就把我们需要传入的数据，保存到我们的sp中。比如如下代码：  

    	SharedPreferences sp=this.getSharedPreferences("TEST",Context.MODE_PRIVATE);
        SharedPreferences.Editor editor=sp.edit();
        editor.putString("key","values");
        editor.putInt("key1",1);
        sp.edit().commit();
        
        //get from sp when we need
        String value= sp.getString("key","");
        Log.i("========",value);  
SharePreference的初始化  
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
1 通过synchronized字段我们可以看到，sp的初始化是线程安全的
2 android首先会通过一个map<String,file>去存储当前安卓系统生成的sp文件。我们在前面传入的TEST name，就会在这个时候进行map的key查找，如果找不到，那么会选择去创建一个sp文件：file = getSharedPreferencesPath(name);  
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
