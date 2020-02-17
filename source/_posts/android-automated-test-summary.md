---
title: Android自动化测试总结
date: 2020-02-16 21:28:12
tags: test
---

# Android自动化测试总结

## 测试金字塔

{% asset_img of-tests.png 测试金字塔，显示了应用的测试套件应包含的三类测试 %}

沿着金字塔逐级向上，从小型测试到大型测试，各类测试的保真度逐级提高，但维护和调试工作所需的执行时间和工作量也逐级增加。因此，您编写的单元测试应多于集成测试，集成测试应多于端到端测试。虽然各类测试的比例可能会因应用的用例不同而异，但我们通常建议各类测试所占比例如下：**小型测试占70%，中型测试占20%，大型测试占10%**。

## 单元测试（小型测试）

**用于验证应用的行为，一次验证一个类。**

**原则（`F.I.R.S.T`）**

**F**ast(快)，单元测试要运行的足够快，单个测试方法一般要立即（一秒之内）给出结果。  
**I**dependent(独立)，测试方法之间不要有依赖（先执行某个测试方法，再执行另一个测试方法才能通过）。  
**R**epeatable（重复），可以在本地或 CI 不同环境（机器上）上反复执行，不会出现不稳定的情况。  
**S**elf-Validating（自验证），单元测试必须包含足够多的断言进行自我验证。  
**T**imely（及时），理想情况下应测试先行，至少保证单元测试应该和实现代码一起及时完成并提交。

除此之外，测试代码应该具备最好的**可读性**和最少的**维护代价**，绝大多数情况下写测试应该就像用**领域特定语言描述一个事实**，甚至**不用经过仔细地思考**。

### 构建本地单元测试

**当需要更快地运行测试而不需要与在真实设备上运行测试关联的保真度和置信度时，可以使用本地单元测试来验证应用的逻辑。**

- 如果测试对`Android`框架有依赖性（特别是与框架建立复杂交互的测试），则最好使用 `Robolectric`添加框架依赖项。

> 例：待测试的类同时依赖`Context`、`Intent`、`Bundle`、`Application`等`Android Framework`中的类时，此时我们可以引入`Robolectric`框架进行本地单元测试的编写。
	
- 如果测试对`Android`框架的依赖性极小，或者如果测试仅取决于我们自己应用的对象，则可以使用诸如`Mockito`之类的模拟框架添加模拟依赖项。([BasicUnitAndroidTest](https://github.com/android/testing-samples/tree/master/unit/BasicUnitAndroidTest))

> 例：待测试的类只依赖`java api`（最理想的情况）,此时对于待测试类所依赖的其他类我们就可以利用`Mockito`框架mock其依赖类，再进行当前类的单元测试编写。([EmailValidatorTest](https://github.com/android/testing-samples/blob/master/unit/BasicSample/app/src/test/java/com/example/android/testing/unittesting/BasicSample/EmailValidatorTest.java))
	
> 例：待测试的类除了依赖`java api`外仅依赖`Android Framework`中`Context`这个类,此时我们就可以利用`Mockito`框架`mock` `Context`类，再进行当前类的单元测试编写。([SharedPreferencesHelperTest](https://github.com/android/testing-samples/blob/master/unit/BasicSample/app/src/test/java/com/example/android/testing/unittesting/BasicSample/SharedPreferencesHelperTest.java)) 

#### 设置测试环境

在`Android Studio`项目中，本地单元测试的源文件存储在`module-name/src/test/java/`中。

在模块的顶级`build.gradle`文件中，将以下库指定为依赖项：

```
    dependencies {
        // Required -- JUnit 4 framework
        testImplementation "junit:junit:$junitVersion"
        // Optional -- Mockito framework
        testImplementation "org.mockito:mockito-core:$mockitoCoreVersion"
        
        // Optional -- Robolectric environment
       testImplementation "androidx.test:core:$xcoreVersion"
       testImplementation "androidx.test.ext:junit:$extJunitVersion"
       testImplementation "org.robolectric:robolectric:$robolectricVersion"
    }   
```

如果单元测试依赖于资源，需要在module的build.gradle文件中启用`includeAndroidResources`选项。然后，单元测试可以访问编译版本的资源，从而使测试更快速且更准确地运行。

```
    android {
        // ...

        testOptions {
            unitTests {
                includeAndroidResources = true
            }
        }
    }
```

```
@RunWith(AndroidJUnit4::class)
@Config(manifest = Config.NONE)
class PeopleDaoTest {
    private lateinit var database: PeopleDatabase

    private lateinit var peopleDao: PeopleDao

    @Before
    fun `create db`() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            PeopleDatabase::class.java
        ).allowMainThreadQueries().build()

        peopleDao = database.peopleDao()
    }

    @Test
    fun `should return empty list when getPeople without inserted data`() {
        val result = peopleDao.getPeople(pageId = 1)

        assertThat(result).isNotNull()
        assertThat(result).isEmpty()
    }
```

如果单元测试包含异步操作时，可以使用[awaitility库](https://github.com/awaitility/awaitility)进行测试；当使用[RxJava](https://github.com/ReactiveX/RxJava)响应式编程库时，可以自定义rule：

```
class RxJavaRule : TestWatcher() {
    override fun starting(description: Description?) {
        super.starting(description)

        RxJavaPlugins.setIoSchedulerHandler {
            Schedulers.trampoline()
        }
        RxJavaPlugins.setNewThreadSchedulerHandler {
            Schedulers.trampoline()
        }
        RxJavaPlugins.setComputationSchedulerHandler {
            Schedulers.trampoline()
        }

        RxAndroidPlugins.setMainThreadSchedulerHandler {
            Schedulers.trampoline()
        }
        RxAndroidPlugins.setInitMainThreadSchedulerHandler {
            Schedulers.trampoline()
        }
    }

    override fun finished(description: Description?) {
        super.finished(description)

        RxJavaPlugins.reset()

        RxAndroidPlugins.reset()
    }
}
```

`TestScheduler`中`triggerActions`的使用。

```
@RunWith(JUnit4::class)
class FilmViewModelTest {
    @get:Rule
    val instantTaskExecutorRule = InstantTaskExecutorRule()
    @get:Rule
    val rxJavaRule = RxJavaRule()

    private val repository = mock(Repository::class.java)

    private val testScheduler = TestScheduler()

    private lateinit var viewModel: FilmViewModel

    @Before
    fun init() {
        viewModel = FilmViewModel(repository)
    }

    @Test
    fun `should return true when loadFilms is loading`() {
        `when`(repository.getPopularFilms(1)).thenReturn(
            Single.just(emptyList<Film>())
                .subscribeOn(testScheduler)
        )

        viewModel.loadFilms(0)

        assertThat(getValue(viewModel.isLoading)).isTrue()
        testScheduler.triggerActions()
        assertThat(getValue(viewModel.isLoading)).isFalse()
    }

    @Test
    fun `should return films list when loadFilms successful`() {
        `when`(repository.getPopularFilms(1)).thenReturn(
            Single.just(
                listOf(
                    Film(123, "", "", "", "", "", "", 1)
                )
            ).subscribeOn(testScheduler)
        )

        viewModel.loadFilms(0)

        assertThat(getValue(viewModel.films)).isNull()
        testScheduler.triggerActions()
        assertThat(getValue(viewModel.films)).isNotNull()
        assertThat(getValue(viewModel.films).size).isEqualTo(1)
    }
}
```

`TestSubscriber`的使用。

```
@RunWith(JUnit4::class)
class WebServiceTest {
    private lateinit var webService: WebService

    private lateinit var mockWebServer: MockWebServer

    @get:Rule
    val instantExecutorRule = InstantTaskExecutorRule()

    @Before
    fun `start service`() {
        mockWebServer = MockWebServer()

        webService = Retrofit.Builder()
            .baseUrl(mockWebServer.url("/"))
            .addConverterFactory(GsonConverterFactory.create())
            .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
            .build()
            .create(WebService::class.java)
    }

    @Test
    fun `should return fim list when getFilms successful`() {
        assertThat(webService).isNotNull()

        enqueueResponse("popular_films.json")

        val testObserver = webService.getPopularFilms(page = 1)
            .map {
                it.data
            }.test()

        testObserver.assertNoErrors()
        testObserver.assertValueCount(1)
        testObserver.assertValue {
            assertThat(it).isNotEmpty()
            assertThat(it[0].id).isEqualTo(297761)
            assertThat(it[1].id).isEqualTo(324668)
            it.size == 2
        }
        testObserver.assertComplete()
        testObserver.dispose()
    }

    @After
    fun `stop service`() {
        mockWebServer.shutdown()
    }

    private fun enqueueResponse(fileName: String) {
        val inputStream = javaClass.classLoader?.getResourceAsStream("api-response/$fileName")
            ?: return
        val source = inputStream.source().buffer()
        val mockResponse = MockResponse()
        mockWebServer.enqueue(
            mockResponse
                .setBody(source.readString(Charsets.UTF_8))
        )
    }
}

```

### 构建插桩单元测试

**插桩单元测试是在物理设备和模拟器上运行的测试，此类测试可以利用`Android`框架`API`。插桩测试提供的保真度比本地单元测试要高，但运行速度要慢得多。因此，我们建议只有在必须针对真实设备的行为进行测试时才使用插桩单元测试。**

#### 设置测试环境

在`Android Studio`项目中，插桩测试的源文件存储在`module-name/src/androidTest/java/`。

在模块的顶级`build.gradle`文件中，将以下库指定为依赖项：

```
    android {
        defaultConfig {
            testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
        }
    }
```

```
    dependencies {
        androidTestImplementation "androidx.test.ext:junit:$extJunitVersion"
        androidTestImplementation "androidx.test:core:$xcoreVersion"
        androidTestImplementation "androidx.test:rules:$rulesVersion"
        // Optional -- Truth library
        androidTestImplementation "androidx.test.ext:truth:$androidxtruthVersion"
        androidTestImplementation "org.mockito:mockito-core:$mockitoCoreVersion"
        androidTestImplementation "org.mockito:mockito-android:$mockitoAndroidVersion"
    }
```

```
@RunWith(AndroidJUnit4::class)
@SmallTest
class FilmDaoTest {
    @get:Rule
    var instantTaskExecutorRule = InstantTaskExecutorRule()

    private lateinit var database: FilmDatabase

    private lateinit var filmDao: FilmDao

    @Before
    fun initDb() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            FilmDatabase::class.java
        ).build()

        filmDao = database.filmData()
    }

    @Test
    fun should_return_film_list_when_getFilms_with_inserted_film_list() {
        filmDao.insert(
            Film(100, "", "", "", "", "", "", 1)
        )
        filmDao.insert(
            Film(101, "", "", "", "", "", "", 1)
        )

        val result = filmDao.getFilms(1)

        assertThat(result).isNotNull()
        assertThat(result).isNotEmpty()
        assertThat(result.size).isEqualTo(2)
        assertThat(result[0].id).isEqualTo(100)
        assertThat(result[0].page).isEqualTo(1)
        assertThat(result[1].id).isEqualTo(101)
    }

    @Test
    fun should_return_film_list_with_size_1_when_getFilms_with_inserted_2_same_film() {
        filmDao.insert(
            Film(100, "", "", "", "", "", "", 1)
        )
        filmDao.insert(
            Film(100, "1223", "111", "", "", "", "", 1)
        )

        val result = filmDao.getFilms(1)

        assertThat(result).isNotNull()
        assertThat(result).isNotEmpty()
        assertThat(result.size).isEqualTo(1)
        assertThat(result[0].id).isEqualTo(100)
        assertThat(result[0].page).isEqualTo(1)
    }

    @Test
    fun should_return_empty_list_when_getFilms_with_deleteAll_called() {
        filmDao.insert(
            Film(100, "", "", "", "", "", "", 1)
        )
        filmDao.deleteAll()

        val newResult = filmDao.getFilms(1)

        assertThat(newResult).isNotNull()
        assertThat(newResult).isEmpty()
    }

    @After
    fun closeDb() = database.close()
}
```

**总结**：

- 基于目前流行的`MVP`、`MVVM`架构设计模式，`MVP`中`Model`层和`Presenter`层尽量不依赖`Android Framework`，`MVVM`中`Model`层和`ViewModel`层尽量不依赖`Android Framework`。
- 类的设计做到单一职责原则,依赖其他类时提供方便mock的方式（例如作为构造方法参数传递），某一个方法依赖其他对象时，小重构该对象作为方法参数传入。
- 方法尽量短小（方法太长时可以利用重构手法在方法中再提取方法）。
- 只覆盖public方法单元测试，privite方法可以间接测试。
- 当依赖`Android Framework API`非常少时，可以采用`Mock Android api`的方式。
- 当严重依赖`Android Framework API`时，引入`Robolectric`库模拟`Android`环境或者放入AndroidTest目录作为插桩单元测试在物理设备上跑。
- 使用`Robolectric`库写本地单元测试时，依赖的某些类的方法调用出问题导致测试`failed`时，可以使用`shadow`类提供默认实现。
- 每条测试采用`Given`、`When`、`Then`的方式进行区分.

 ```
@Test
public void should_do_something_if_some_condition_fulfills() {
    // Given 设置前置条件

    // When 执行被测方法

    // Then 验证方法结果
}
```

## 集成测试（中型测试）

**用于验证模块内堆栈级别之间的交互或相关模块之间的交互**

- 如果应用使用了用户不直接与之交互的组件（如`Service`或`ContentProvider`），应验证这些组件在应用中的行为是否正确。

### 设置测试环境

**参考插桩单元测试环境设置**

#### Service测试

- **利用`ServiceTestRule`，可在单元测试方法运行之前启动服务，并在测试完成后关闭服务。**
- **`ServiceTestRule`类不支持测试`IntentService`对象。如果需要测试`IntentService`对象，可以应将逻辑封装在一个单独的类中，并创建相应的单元测试。**

```
@MediumTest
@RunWith(AndroidJUnit4.class)
public class LocalServiceTest {
    @Rule
    public final ServiceTestRule mServiceRule = new ServiceTestRule();

    @Test
    public void testWithBoundService() throws TimeoutException {
        // Create the service Intent.
        Intent serviceIntent =
                new Intent(getApplicationContext(), LocalService.class);

        // Data can be passed to the service via the Intent.
        serviceIntent.putExtra(LocalService.SEED_KEY, 42L);

        // Bind the service and grab a reference to the binder.
        IBinder binder = mServiceRule.bindService(serviceIntent);

        // Get the reference to the service, or you can call public methods on the binder directly.
        LocalService service = ((LocalService.LocalBinder) binder).getService();

        // Verify that the service is working correctly.
        assertThat(service.getRandomInt(), is(any(Integer.class)));
    }
}
```

#### ContentProvider的测试

**使用`ProviderTestRule`**

```
 @Rule
 public ProviderTestRule mProviderRule =
     new ProviderTestRule.Builder(MyContentProvider.class, MyContentProvider.AUTHORITY).build();

 @Test
 public void verifyContentProviderContractWorks() {
     ContentResolver resolver = mProviderRule.getResolver();
     // perform some database (or other) operations
     Uri uri = resolver.insert(testUrl, testContentValues);
     // perform some assertions on the resulting URI
     assertNotNull(uri);
 }
```

```
 @Rule
 public ProviderTestRule mProviderRule =
     new ProviderTestRule.Builder(MyContentProvider.class, MyContentProvider.AUTHORITY)
         .setDatabaseCommands(DATABASE_NAME, INSERT_ONE_ENTRY_CMD, INSERT_ANOTHER_ENTRY_CMD)
         .build();

 @Test
 public void verifyTwoEntriesInserted() {
     ContentResolver mResolver = mProviderRule.getResolver();
     // two entries are already inserted by rule, we can directly perform assertions to verify
     Cursor c = null;
     try {
       c = mResolver.query(URI_TO_QUERY_ALL, null, null, null, null);
       assertNotNull(c);
       assertEquals(2, c.getCount());
     } finally {
       if (c != null && !c.isClosed()) {
         c.close();
       }
     }
 }
```

- `Android`没有为`BroadcastReceiver`提供单独的测试用例类。要验证 `BroadcastReceiver`是否正确响应，可以测试向其发送`Intent`对象的组件。或者，可以通过调用`ApplicationProvider.getApplicationContext()`来创建`BroadcastReceiver`的实例，然后调用要测试的`BroadcastReceiver`方法（通常是`onReceive()`方法）。

## 端到端测试（大型测试）

**用于验证跨越了应用的多个模块的用户操作流程**

**界面测试的一种方法是直接让测试人员对目标应用执行一系列用户操作，并验证其行为是否正常。不过，这种人工方法会非常耗时、繁琐且容易出错。一种更高效的方法是编写界面测试，以便以自动化方式执行用户操作。自动化方法可以以可重复的方式快速可靠地运行测试。**

### 设置测试环境

```
    dependencies {
        androidTestImplementation "androidx.test.ext:junit:$extJunitVersion"
        androidTestImplementation "androidx.test:core:$xcoreVersion"
        androidTestImplementation "androidx.test:rules:$rulesVersion"
        // Optional -- Truth library
        androidTestImplementation "androidx.test.ext:truth:$androidxtruthVersion"
        androidTestImplementation "org.mockito:mockito-core:$mockitoCoreVersion"
        androidTestImplementation "org.mockito:mockito-android:$mockitoAndroidVersion"
         // Optional -- UI testing with Espresso
        androidTestImplementation "androidx.test.espresso:espresso-core:$espressoVersion"
        androidTestImplementation "androidx.test.espresso:espresso-intents:$espressoVersion"
        // Optional -- UI testing with UI Automator
        androidTestImplementation "androidx.test.uiautomator:uiautomator:$uiautomatorVersion"
    }
```

- **涵盖单个应用的界面测试**：这种类型的测试可验证目标应用在用户执行特定操作或在其 `Activity` 中输入特定内容时的行为是否符合预期。它可让您检查目标应用是否返回正确的界面输出来响应应用 `Activity` 中的用户交互。诸如 `Espresso` 之类的界面测试框架可让您以编程方式模拟用户操作，并测试复杂的应用内用户交互。([espresso测试单个应用的界面例子](https://github.com/android/testing-samples/tree/master/ui/espresso))

- **涵盖多个应用的界面测试**：这种类型的测试可验证不同用户应用之间交互或用户应用与系统应用之间交互的正确行为。例如，您可能想要测试相机应用是否能够与第三方社交媒体应用或默认的 `Android` 相册应用正确分享图片。支持跨应用交互的界面测试框架（如 `UI Automator`）可让您针对此类场景创建测试。（[uiautomator测试多个应用的界面](https://github.com/android/testing-samples/tree/master/ui/uiautomator/BasicSample)）

[参考例子testing-samples](https://github.com/android/testing-samples)
