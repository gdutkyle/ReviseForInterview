# 从SQLCipher到WCDB  
## 为什么要使用WCDB替换已有的SQLCipher？   
WCDB实际上是基于SQLCipher上设计的，当初我们项目接入wcdb的第一个初衷是，想要看看我们的apk能不能小点，后来发现其实不会。因为wcdb最基本的就是sqlcipher，所以，如果只使用基本的功能，那么是不会对基本sql大小有影响的。但是，如果我们需要使用到数据库分词icu（wcdb对应mmicu），那么确实是可以把我们的apk缩小大概10m左右，这个我们后面再讲。第二个初衷是，wcdb提供了比较强大的数据库备份和恢复的功能，我们的项目在给用户使用的时候，还是存在数据库损坏的情况。所以，鉴于这两点，我们将原来的sqlcipher，替换成了wcdb。  
## wcdb的接入  
基于sqlcipher的wcdb，在接入到我们的项目中出乎意料的简单。我们接下来看下具体的步奏  
### 第一步：引入依赖  
在app的build.gradle中，添加

    dependencies {  
    compile 'com.tencent.wcdb:wcdb-android:1.0.0'
	}
### 第二步：选择需要使用的架构  
我们工程使用的是armeabi-v7a，如果有需要，可以自由选择当前工程的实际需要  

    android {
    	defaultConfig {
       		ndk {
            abiFilters 'armeabi-v7a'
        	}
    	}
	}  
### 第三步：开始替换  
替换其实很简答，只要把工程中`android.database.* 改为 com.tencent.wcdb.*`，`android.database.sqlite.* 改为 com.tencent.wcdb.database.*`。  
### 第四步：开始数据库的迁移  

    private PersonalDatabase() {
        	String passphrase = getDBKey();//生成加密数据库的key
            String dbPath = ECContextParameter.getDbPath() + DB_NAME;//db所在的path
            SQLiteCipherSpec cipher = new SQLiteCipherSpec()  // 加密描述对象
                    .setPageSize(1024)        // SQLCipher 默认 Page size 为 1024
                    .setSQLCipherVersion(3);  // 1,2,3 分别对应 1.x, 2.x, 3.x 创建的 SQLCipher 数据库,当前版本用的是3.X版本

            SQLiteOpenHelper helper = new PersonalDatabaseOpenHelper(getContext(), dbPath, DB_VERSION, passphrase, cipher);
            encryptedDB = helper.getWritableDatabase();
    }  
迁移的步奏在上面有详细的代码，这里不做赘述。
