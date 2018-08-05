# Android Architecture Components之ViewModel 解析  
###一 Why ViewModel？  
根据官方文档我们知道，ViewModel是用来存储或者管理一些跟UI相关的数据，在生命周期发生改变的时候，ViewModel中的数据不会发生改变。ViewModel中的数据，在一些配置发生改变的时候，也能存活。比如当我们旋转手机屏幕的时候。  

众所周知，我们的UI控制器的生命周期是Android framework控制的，Android framework会在在某些特定的情况下，摧毁或者重建我们的UI控制器，而这些动作都不是我们可以控制的。而当系统决定摧毁或者重建我们的UI控制器的时候，我们设置的，跟这些UI相关的数据，都会被重新初始化。这个时候，你会问，不是还有onSaveInstanceState()方法吗？但是这个方法，只能暂存一些很小量的数据，而且这些数据是必须支持序列化和反序列化的，也就是说，你可能从某个地方获取了一个大量的users列表，或者bitmap数据，但是很不幸，当系统决定重新重建你的UI的时候，这一切都将不复存在。  

另一个问题是，UI控制器通常需要异步去获取数据，这个需要一些时间去等待数据的返回。这样就会造成，我们在异步回调的地方，需要确定当UI被destory的时候，数据会被及时清除，避免造成内存泄漏。这个控制逻辑就会造成，当我们需要重建UI控制器的时候，这些数据需要重新去获取，这样就会造成资源的浪费。  

###二 VieModel的生命周期  
什么都别说了，先放图  
![图片来自网络](https://i.imgur.com/G4W4G8E.png)  
我们从图中可以看出，ViewModel的生命周期是贯穿整个Activity的生命周期的，也就是在Activity没有Finished（注意，不是finish）之前，viewModel都是存活的。所以，我们就可以确保，无论Activity的UI是怎么折腾的，怎么摧毁又重建的，ViewModel中的数据都不会被清空。  

###三 Fragments之间分享数据  
想象一下，我们有一些很典型的业务场景，在一个Activity下，我们绑定了两个fragment，这个时候，FragmentA
中某个数据data发生了变化，需要通知到FragmentB。这对于我们两说，将会是一个非常繁琐的问题，如何保证两个Fragmet都被add进去了，如果控制内存不泄漏，如果确保两个数据即时有效。  
如果我们使用了ViewModel，那么将会是多么愉快的事情。我们看下官方的demo  

    public class SharedViewModel extends ViewModel {
    	private final MutableLiveData<Item> selected = new MutableLiveData<Item>();

    	public void select(Item item) {
       		selected.setValue(item);
    	}

    	public LiveData<Item> getSelected() {
        	return selected;
   		}
	}


    public class MasterFragment extends Fragment {
    	private SharedViewModel model;
    	public void onCreate(Bundle savedInstanceState) {
        	super.onCreate(savedInstanceState);
        	model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        	itemSelector.setOnClickListener(item -> {
            model.select(item);
        	});
    	}
    }
    public class DetailFragment extends Fragment {
    	public void onCreate(Bundle savedInstanceState) {
        	super.onCreate(savedInstanceState);
        	SharedViewModel model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        	model.getSelected().observe(this, item -> {
           	// Update the UI.
        	});
    	}
	}  
从上面的demo可以看出MasterFragment 的model发生变更的时候，会同步将DetailFragment的UI进行更新。 
采用这种方式，有以下几个好处  
1 Activity不需要知道任何的数据，不需要做任何的数据  
2 Fragment不需要知道双方对于这个SharedViewModel的用途，如果其中一个Fragment消失了，另一个Fragment也不会有影响  
3 每个Fragment都有自己的生命周期，并且两者不会相互影响。如果其中一个代替了另一个，也不会对UI产生任何影响 
###四 CursorLoader已经可以弃用？  
我们以前经常使用loaderManager+CursorLoader的方式，对数据库进行数据监听，然后返回展示数据库服务条件的数据。当数据库中的某个数据发生变化的时候，列表能够马上监听的到，并刷新我们的UI。这个流程用下图来显示就是  
![图片来自网络](https://i.imgur.com/2x0HIOR.png)  
那么我们使用ViewModel，就不会再需要那么复杂了。我们使用LiveData+ViewModel+Room的方式，就可以将上面的流程更直观的开发出来  
![图片来自网络](https://i.imgur.com/5QRHdH4.png)

如果大家有需要这个类似的demo，我开发了个小demo，放在github上了  
[https://github.com/gdutkyle/AndroidAComponent](https://github.com/gdutkyle/AndroidAComponent "github地址")  
