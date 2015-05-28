# AndroidMvc Framework
## Features
  - Clear division between view logic and business logic
  - Event driven
  - Easy testing for business logic on JVM without Android dependency
  - Dependency injection - easier to mock
  - Clear navigation logic and testable
  - Automatically save restore instance state

## Overview
AndroidMvc is event driven based on event bus. Controllers can be injected into views by fields annotated by @Inject. When views need to handle business logic they will issue commands to Controllers. Once controllers
finish processing the command they send the result back to views through event bus. Then all views
subscribed to that event will receive the result and respond to it.

To isolate events between different layers, there are 3 event buses pre-setup in the framework:
- **EventBusC2V** (Controllers to Views): One way event bus routing events from controllers to views. Events sent to views will be guaranteed to be run on Android UI thread by the framework.
- **EventBusC2C** (Controllers to Controllers): Routes events among controllers. Events will be received on the same thread who send them.
- **EventBusV2V** (Views to views): Routes events among views. Events will be received on the same thread who send them.

#### View
All views should be **as lean as possible** that they are only responsible to capture user interaction and display data sent back from controllers. Therefore, business logic can be maximally abstracted away from views into controllers. As a result, more unit testings can be done on controllers. At a high level, any components of Android framework could be considered as views including activities, fragments, widgets or even services and etc, because responsibilities of all components above are just to capture user interactions and present the data.

#### Controller
Controllers manage business logic, how to retrieve data, calculate and present data. In this MVC design, all controllers are **SINGLETON** application wide. 

#### Model
Models in AndroidMVC design represent the state of controllers. So each controller has at most one
model to represent the state of the controller. The state of the controller will be automatically saved when
Android saveInstanceState and get restored when Android restoreInstanceState.


## Using AndroidMvc
Let's take a simple app counting number as an example. The counter app has two navigation locations: 
1. LocationA: presented by FragmentA
   * One text view to display the current count in number. Updated by event OnCounterUpdated
   * Two buttons which increment and decrement count **on click**.
   * An nested fragment with a TextView to display count in English. Updated by event OnCounterUpdated too
   * An button to show advance view which results in navigating to LocationB
   * Shares the result of counting updated by LocationB
2. LocationB: presented by FragmentB
   * One text view to display the current count in number. Updated by event OnCounterUpdated
   * Two buttons which increment and decrement count **continuously on hold**.
   * Shares the result of counting updated by LocationB

Below is how to use AndroidMvc framework to implement and test the app including navigation. Note the code below doesn't show all code. To see more details check the sample project in the app - Simple under samples subfolder.

##### 1. Extend MvcActivity for the single Activity
````java
/**
 * Single activity for the app
 */
public class MainActivity extends MvcActivity {
    /**
     * Define how to map navigation location id to full screen fragments
     * @param locationId The location id in string
     * @return The class of the fragment representing the navigation locations
     */
    @Override
    protected Class<? extends MvcFragment> mapNavigationFragment(String locationId) {
        switch (locationId) {
            case "LocationA":
                return FragmentA.class;
            case "LocationB":
                return FragmentB.class;
            default:
                return null;
        }
    }

    /**
     * Define the delegate fragment for the activity
     * @return
     */
    @Override
    protected Class<? extends DelegateFragment> getDelegateFragmentClass() {
        return ContainerFragment.class;
    }

    /**
     * Container fragment extends DelegateFragment would be the root container fragments to swap
     * full screen fragments inside it on navigation.
     */
    public static class ContainerFragment extends DelegateFragment {
        /**
         * What to do when app starts for the first time
         */
        @Override
        protected void onStartUp() {
            //Navigate to location a when app starts for the first time by navigation controller
            //here we navigate to LocationA which result in load FragmentA mapped by the
            //the method mapNavigationFragment above
            getNavigationController().navigateTo(this, "LocationA");
        }
    }
}
````
##### 2. Create FragmentA to present "LocationA"
````java
public class FragmentA extends MvcFragment {
    /**
     * @return Layout id used to inflate the view of this MvcFragment.
     */
    @Override
    protected int getLayoutResId() {
        return R.layout.fragment_a;
    }
}
````
##### 3. Create a controller contract and the model it's managing
````java
package com.shipdream.lib.android.mvc.samples.simple.controller;

/**
 * Define controller contract and its events. And specify which model it manages by binding the 
 * model type.
 */
public interface CounterController extends BaseController<CounterModel> {
    /**
     * Increment count and will raise {@link EventC2V.OnCounterUpdated}
     * @param sender Who requests this action
     */
    void increment(Object sender);

    /**
     * Decrement count and will raise {@link EventC2V.OnCounterUpdated}
     * @param sender Who requests this action
     */
    void decrement(Object sender);

    /**
     * Method to convert number to english
     * @param number
     * @return
     */
    String convertNumberToEnglish(int number);

    /**
     * Namespace the events for this controller by nested interface so that all its events would
     * be referenced as CounterController.EventC2V.BlaBlaEvent
     */
    interface EventC2V {
        /**
         * Event to notify views counter has been updated
         */
        class OnCounterUpdated extends BaseEventC2V {
            private final int count;
            private final String countInEnglish;
            public OnCounterUpdated(Object sender, int count, String countInEnglish) {
                super(sender);
                this.count = count;
                this.countInEnglish = countInEnglish;
            }

            public int getCount() {
                return count;
            }

            public String getCountInEnglish() {
                return countInEnglish;
            }
        }
    }
}
````
##### 3. Implement the controller
**Note that, to allow AndroidMvc to find the default implementation of injectable object, the implementation class must be under the sub-package "internal" which resides in the same parent package as the interface and the name must be [InterfaceName]Impl.** For this example, say CounterController is under package samples.simple.controller the  implementation must be named as CounterControllerImpl and placed under package samples.simple.controller.internal
````java
/**
 * Note the structure of the package name. It is in a subpackage(internal) sharing the same parent 
 * package as the controller interface CounterController
 */
package com.shipdream.lib.android.mvc.samples.simple.controller.internal;

/**
 * Note the class name is [CounterController]Impl.
 */
public class CounterControllerImpl extends BaseControllerImpl<CounterModel> implements CounterController{
    /**
     * Just return the class type of the model managed by this controller
     * @return
     */
    @Override
    protected Class<CounterModel> getModelClassType() {
        return CounterModel.class;
    }

    @Override
    public void increment(Object sender) {
        int count = getModel().getCount();
        getModel().setCount(++count);
        //Post controller to view event to views
        postC2VEvent(new EventC2V.OnCounterUpdated(sender, count, convertNumberToEnglish(count)));
    }

    @Override
    public void decrement(Object sender) {
        int count = getModel().getCount();
        getModel().setCount(--count);
        //Post controller to view event to views
        postC2VEvent(new EventC2V.OnCounterUpdated(sender, count, convertNumberToEnglish(count)));
    }

    @Override
    public String convertNumberToEnglish(int number) {
        if (number < -3) {
            return "Less than negative three";
        } else  if (number == -3) {
            return "Negative three";
        } else  if (number == -2) {
            return "Negative two";
        } else  if (number == -1) {
            return "Negative one";
        } else if (number == 0) {
            return "Zero";
        } else if (number == 1) {
            return "One";
        } else if (number == 2) {
            return "Two";
        } else if (number == 3) {
            return "Three";
        } else {
            return "Greater than three";
        }
    }
}
````

##### 4. Inject Controller into Views, setup views and handle C2V events from controllers
````java
public class FragmentA extends MvcFragment {
	@Inject
    private CounterController counterController;

    private TextView display;
    private Button increment;
    private Button decrement;

    /**
     * @return Layout id used to inflate the view of this MvcFragment.
     */
    @Override
    protected int getLayoutResId() {
        return R.layout.fragment_a;
    }

    /**
     * Lifecycle similar to onViewCreated by with more granular control with an extra argument to 
     * indicate why this view is created: 1. first time created, or 2. rotated or 3. restored
     * @param view The root view of the fragment
     * @param savedInstanceState The savedInstanceState when the fragment is being recreated after
     *                           its enclosing activity is killed by OS, otherwise null including on
     *                           rotation
     * @param reason Indicates the {@link Reason} why the onViewReady is called.
     */
    @Override
    public void onViewReady(View view, Bundle savedInstanceState, Reason reason) {
        super.onViewReady(view, savedInstanceState, reason);

        display = (TextView) view.findViewById(R.id.fragment_a_counterDisplay);
        increment = (Button) view.findViewById(R.id.fragment_a_buttonIncrement);
        decrement = (Button) view.findViewById(R.id.fragment_a_buttonDecrement);

        increment.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                counterController.increment(v);
            }
        });

        decrement.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                counterController.decrement(v);
            }
        });
        
        updateCountDisplay(counterController.getModel().getCount());
    }

    /**
     * Callback when the fragment is popped out by back navigation
     */
    @Override
    protected void onPoppedOutToFront() {
        super.onPoppedOutToFront();
        updateCountDisplay(counterController.getModel().getCount());
    }

    //Define event handler by method named as onEvent with single parameter of the event type
    //to respond event CounterController.EventC2V.OnCounterUpdated
    private void onEvent(CounterController.EventC2V.OnCounterUpdated event) {
        updateCountDisplay(event.getCount());
    }

    /**
     * Update the text view of count number
     * @param count The number of count
     */
    private void updateCountDisplay(int count) {
        display.setText(String.valueOf(count));
    }
}
````
##### 5. Unit tests on controllers
As discussed before, business logic should be decoupled from view(Android components) and abstracted to controllers, then we can pretty much test most logic just on controllers without dependencies of any Android components. Views just need to make sure data carried back from controllers are displayed correctly. Whether or not the data is processed correctly is completely controllers' responsibilities that is what is being tested here.

````java
public class TestCounterController {
	...other dependencies are omitted here
    
    private CounterController counterController;

    @Before
    public void setUp() throws Exception {
    	...other dependencies are omitted here
        
        //create instance of CounterController
        counterController = new CounterControllerImpl();
        counterController.init();
    }

    @Test
    public void increment_should_post_counter_update_event_with_incremented_value() {
        //1. Prepare
        //prepare event monitor
        class Monitor {
            void onEvent(CounterController.EventC2V.OnCounterUpdated event) {
            }
        }
        Monitor monitor = mock(Monitor.class);
        eventBusC2V.register(monitor);

        //mock controller model for count value
        int value = new Random().nextInt();
        CounterModel counterModel = mock(CounterModel.class);
        when(counterModel.getCount()).thenReturn(value);
        //Bind the mock model to the controller
        counterController.bindModel(this, counterModel);

        //2. Act
        counterController.increment(this);

        //3. Verify
        ArgumentCaptor<CounterController.EventC2V.OnCounterUpdated> updateEvent
                = ArgumentCaptor.forClass(CounterController.EventC2V.OnCounterUpdated.class);
        //event should be fired once
        verify(monitor, times(1)).onEvent(updateEvent.capture());
        //event should carry incremented value
        Assert.assertEquals(value + 1, updateEvent.getValue().getCount());
    }
}
````
##### 5. Navigation
Instead creating, replacing or popping full screen fragments by FragmentManager of Android Activity, AndroidMvc provides NavigationController to manage navigation. Therefore, navigation logic can be abstracted out from View layer. To make navigation easier to be tested, we can inject NavigationController to CounterController and then test the model of NavigationController to verify if navigation location is changed as expected.
##### 5.1. Add two methods to CounterController to wrap navigation logic
````java
public interface CounterController extends BaseController<CounterModel> {
	... other methods
    
	/**
     * Navigate to LocationB by {@link NavigationController}to show advance view that can update
     * count continuously by holding buttons.
     * @param sender
     */
    void goToAdvancedView(Object sender);

    /**
     * Navigate back to LocationA by {@link NavigationController}to show basic view from LocationB
     * @param sender
     */
    void goBackToBasicView(Object sender);
    
	... other methods
}
````
##### 5.2. Inject NavigationController to CounterControllerImpl and implement navigation methods
````java
public class CounterControllerImpl extends BaseControllerImpl<CounterModel> implements CounterController{
	... other methods
    
    @Inject
    NavigationController navigationController;

    @Override
    public void goToAdvancedView(Object sender) {
        navigationController.navigateTo(sender, "LocationB");
    }

    @Override
    public void goBackToBasicView(Object sender) {
        navigationController.navigateBack(sender);
    }
    
    ... other methods
}
````
##### 5.3. Invoke CounterController methods wrapping navigation in views
````java
public class FragmentA extends MvcFragment {
	...
    
    @Override
    public void onViewReady(View view, Bundle savedInstanceState, Reason reason) {
        super.onViewReady(view, savedInstanceState, reason);
		
        ...
        
        buttonShowAdvancedView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //Use counterController to manage navigation to make navigation testable
                counterController.goToAdvancedView(v);
                //Or we can use NavigationController directly though it's harder to unit test on
                //controller level.
                //example:
                //navigationController.navigateTo(v, "LocationB");
            }
        });
		
        ...
    }
    
    ...
}

public class FragmentB extends MvcFragment {
	...
    
    @Override
    public boolean onBackButtonPressed() {
        //Use counterController to manage navigation back make navigation testable
        counterController.goBackToBasicView(this);
        //Return true to not pass the back button pressed event to upper level handler.
        return true;
        //Or we can let the fragment manage back navigation back automatically where we don't
        //override this method which will call NavigationController.navigateBack(Object sender)
        //automatically
    }
    
    ...
}
````
##### 5.4. Able to test navigation on CounterController
````java
    @Test
    public void should_navigate_to_locationB_when_go_to_advance_view_and_back_to_locationA_after_go_to_basic_view() {
        //Prepare
        NavigationController navigationController = ((CounterControllerImpl) counterController).navigationController;
        NavigationController.Model navModel = navigationController.getModel();
        //App has not navigated to anywhere, current location should be null
        Assert.assertNull(navModel.getCurrentLocation());
        //Simulate navigating to location A
        navigationController.navigateTo(this, "LocationA");
        //Verify: location should be changed to LocationA
        Assert.assertEquals(navModel.getCurrentLocation().getLocationId(), "LocationA");

        //Act: CounterController now goes to advanced view underlining logic is navigating to locationB
        counterController.goToAdvancedView(this);

        //Verify: Current location should be LocationB
        Assert.assertEquals(navModel.getCurrentLocation().getLocationId(), "LocationB");

        //Act: CounterController now goes back to basic view underlining logic is navigating back to locationA
        counterController.goBackToBasicView(this);

        //Verify: Current location should be back to LocationA
        Assert.assertEquals(navModel.getCurrentLocation().getLocationId(), "LocationA");
    }
````

## Other features
#### 1. Dependency Injection

##### @Inject

The framework currently only support field injection. To inject an object, use @Inject to annotate fields and then inject the object with those fields by
````java
AndroidMvc.graph().inject(ObjectToBeInjected)
````
All injected objects will be reference counted. This is because
the graph will automatically save and restore injected objects implementing StateManaged. For performance reasons the injected but not anymore referenced objects don't need to be saved and restored. So don't forget to call
````java
AndroidMvc.graph().release(ObjectBeenInjected)
````
to dereference them. Fortunately, all MvcFragment will do the injection and releasing in their
Android lifecycle - onCreate and onDestroy. So we don't need to do this manually for fragments.

##### @Provides
To provide the instance to inject the
fields, there are 2 ways.

1. **Name pattern**. The implementation of the injecting class will be located by file structure and
file name. A default implementation will be looked for in the sub folder - internal from the parent
folder of the interface and the name of the file should be [InterfaceName]Impl. For example, we
define a controller LoginController in folder /controllers/LoginController then the implementation
should be placed under /controllers/internal and named as LoginControllerImpl. So the implementation
should be at /controllers/internal/LoginControllerImpl

2. **Register providers** by extending a Component and annotate the methods providing the instance by
**@Provides**. This can be used to replace the implementation located by the name pattern described
above for the unit testing. Also the component can be unregistered from **@Singleton** can
be used if singleton is required. Beware singleton is relative to the ScopeCache associated to the
Component. So to make the instance absolute singleton, either guarantee the Component is the unique
or the ScopeCache associated to the Component is unique. In other words, if the component is
recreated or the cache of it is recreated it new instance will be provided and cached sharing the
same time span as the component or its cache again.


### 2. Unit testing on asynchronous actions, e.g. Http requests
Below is an example to consume a public weather API from [OpenWeatherMap](http://openweathermap.org/api). To be able to test controller without real http communication, the http request can be abstracted into a service interface. The service interface is injected into controllers. Then in real implementation of the service interface we send http request by http client while in controller testings we mock the service to provide mock data. 

See more details in the sample project - Node

**Note that, BaseMvcControllerImpl provides protected methods to run actions asynchronously. The ExecutorService is injected into controllers. By default, AndroidMvc framework automatically injects with an implementation running tasks on non-main thread. Whereas in unit tests we can override the injection with an implementation runs the task on the same thread as the caller's so that the asynchronous actions can be tested easier.**
````java
/**
 * Run async task on the default ExecutorService injected as a field of this class. Exceptions
 * occur during running the task will be handled by the given {@link AsyncExceptionHandler}.
 *
 * @param sender                who initiated this task
 * @param asyncTask             task to execute
 * @param asyncExceptionHandler error handler for the exception during running the task
 * @return the reference of {@link AsyncTask} that can be used to query its state and cancel it.
 */
protected AsyncTask runAsyncTask(Object sender, AsyncTask asyncTask, AsyncExceptionHandler asyncExceptionHandler)
````

##### 1. Define http service interface
````java
package com.shipdream.lib.android.mvc.samples.note.service.http;

public interface WeatherService {
    /**
     * Get weathers of the cities with the given ids
     * @param ids Ids of the cities
     * @return The response
     */
    WeatherListResponse getWeathers(List<Integer>ids) throws IOException;
}
````
##### 2. Send and consume real http service in implementation
````java
/**
 * Note the package structure which is under internal subpackage sharing the same parent package as 
 * WeatherService as above
 */
package com.shipdream.lib.android.mvc.samples.note.service.http.internal;

public class WeatherServiceImpl implements WeatherService{
    private HttpClient httpClient;
    private Gson gson;

    public WeatherServiceImpl() {
        httpClient = new DefaultHttpClient();
        gson = new Gson();
    }

    @Override
    public WeatherListResponse getWeathers(List<Integer> ids) throws IOException {
        String idsStr = "";
        for (Integer id : ids) {
            if (!idsStr.isEmpty()) {
                idsStr += ", ";
            }
            idsStr += String.valueOf(id);
        }
        String url = String.format("http://api.openweathermap.org/data/2.5/group?id=%s&units=metric",
                URLEncoder.encode(idsStr, "UTF-8"));
        HttpGet get = new HttpGet(url);
        HttpResponse resp = httpClient.execute(get);
        String responseStr = EntityUtils.toString(resp.getEntity());
        return gson.fromJson(responseStr, WeatherListResponse.class);
    }
}
````
##### 3. Inject the http service into WeatherControllerImpl
````java
public class WeatherControllerImpl extends BaseControllerImpl <WeatherModel> implements
        WeatherController{
	....
    
    @Inject
    private WeatherService weatherService;
    
    //consume the service and fetch weathers
    //...
}
````
##### 4. In controller unit test, override injection of ExecutorService with implementation running actions on the same thread as the caller's
Code below is partial implementation, see sample Note in the project for more details.
````java
public class TestWeatherController extends TestControllerBase<WeatherController> {
@Override
protected void registerDependencies(MvcGraph mvcGraph) {
	...
    
	//Setup mock executor service mock that runs task on the same thread.
	executorService = mock(ExecutorService.class);
	doAnswer(new Answer() {
		@Override
		public Object answer(InvocationOnMock invocation) throws Throwable {
			Runnable runnable = (Runnable) invocation.getArguments()[0];
			runnable.run();
			return null;
		}
	}).when(executorService).submit(any(Runnable.class));

    //Register the injecting component to mvcGraph to override the implementation being injected
    //to controllers
	TestComp testComp = new TestComp();
	testComp.testNoteController = this;
	mvcGraph.register(testComp);
}
}
````

##### 5. Test if WeatherController sends successful event with good http response
What to mock
1. Http Service to provide good response
2. Event monitor to subscribe to the successful event
So when WeatherController#updateAllCities(Object) is called, we can verify whether the mocked monitor receives the successful event.

````java
@Test
public void shouldRaiseSuccessEventForGoodUpdateWeathers() throws IOException {
	//---Arrange---
	//Define a subscriber class
	class Monitor {
		public void onEvent(WeatherController.EventC2V.OnWeathersUpdated event) {
		}
		public void onEvent(WeatherController.EventC2V.OnWeathersUpdateFailed event) {
		}
	}
	Monitor monitor = mock(Monitor.class);
    //Subscribe to eventBus
	eventBusC2V.register(monitor);

    //Weather service mock prepares a good response
	WeatherListResponse responseMock = mock(WeatherListResponse.class);
	when(weatherServiceMock.getWeathers(any(List.class))).thenReturn(responseMock);

	//---Action---
	controllerToTest.updateAllCities(this);

	//---Verify---
	//Success event should be raised
	ArgumentCaptor<WeatherController.EventC2V.OnWeathersUpdated> eventSuccess
			= ArgumentCaptor.forClass(WeatherController.EventC2V.OnWeathersUpdated.class);
	verify(monitor, times(1)).onEvent(eventSuccess.capture());
	//Failed event should not be raised
	ArgumentCaptor<WeatherController.EventC2V.OnWeathersUpdateFailed> eventFailure
			= ArgumentCaptor.forClass(WeatherController.EventC2V.OnWeathersUpdateFailed.class);
	verify(monitor, times(0)).onEvent(eventFailure.capture());
}
````

##### 6. **Test if WeatherController sends failed event with bad http response**
What to mock
1. Http Service to provide bad response
2. Event monitor to subscribe to the failed event
So when WeatherController#updateAllCities(Object) is called, we can verify whether the mocked monitor receives the failed event.
````java
@Test
public void shouldRaiseFailEventForNetworkErrorToUpdateWeathers() throws IOException {
	//---Arrange---
	//Define a subscriber class
	class Monitor {
		public void onEvent(WeatherController.EventC2V.OnWeathersUpdated event) {
		}
		public void onEvent(WeatherController.EventC2V.OnWeathersUpdateFailed event) {
		}
	}
	Monitor monitor = mock(Monitor.class);
    //Subscribe to eventBus
	eventBusC2V.register(monitor);
    
	//Weather service mock prepares a bad response 
    //by throwing an exception when getting the weather data
	when(weatherServiceMock.getWeathers(any(List.class))).thenThrow(new IOException());

	//---Action---
	controllerToTest.updateAllCities(this);

	//---Verify---
	//Success event should not be raised
	ArgumentCaptor<WeatherController.EventC2V.OnWeathersUpdated> eventSuccess
			= ArgumentCaptor.forClass(WeatherController.EventC2V.OnWeathersUpdated.class);
	verify(monitor, times(0)).onEvent(eventSuccess.capture());
	//Failed event must be raised
	ArgumentCaptor<WeatherController.EventC2V.OnWeathersUpdateFailed> eventFailure
			= ArgumentCaptor.forClass(WeatherController.EventC2V.OnWeathersUpdateFailed.class);
	verify(monitor, times(1)).onEvent(eventFailure.capture());
}
````

### 3. Custom mechanism to automatically save/restore models of controllers
By default, AndroidMvc uses GSON to serialize and deserialize models of controllers automatically. In general uses the performance is acceptable. For example, on rotation, as long as the models are not very large, the frozen time of the rotation would be between 200ms and 300ms.

If we need to provide more optimized mechanism to do so in case there are large models taking long to be serialized and deserialized by GSON, custom StateKeeper can be set to provide alternative save/restore implementation. For Android, Parcelable is the best performed mechanism to save/restore state but it is not fun and error prone. Fortunately, there a handy library [Parceler](https://github.com/johncarl81/parceler) from another developer does this automatically. In the example below, we tried this library to implement custom StateKeeper to save/restore state by Parcelables automatically. The best place to set the custom StateKeeper is the Application#onCreate(). 

Check out more details in the sample code - Note

````java
public class NoteApp extends Application {
    @Override
    public void onCreate() {
        super.onCreate();

        AndroidMvc.setCustomStateKeeper(new AndroidStateKeeper() {
            @SuppressWarnings("unchecked")
            @Override
            public Parcelable saveState(Object state, Class type) {
                /**
                 * Use parcelable to save all states.
                 */
                return Parcels.wrap(state);
                //type of the state can be used as a filter to handle some state specially
                //if (type == BlaBlaType) {
                //    special logic to save state
                //}
            }

            @SuppressWarnings("unchecked")
            @Override
            public Object getState(Parcelable parceledState, Class type) {
                /**
                 * Use parcelable to restore all states.
                 */
                return Parcels.unwrap(parceledState);
                
                //type of the state can be used as a filter to handle some state specially
                //if (type == BlaBlaType) {
                //    special logic to restore state
                //}
            }
        });
    }
}
````