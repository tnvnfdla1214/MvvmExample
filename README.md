# Mvvm+acc 시작해보기 ( Room + viewModel + LiveData + RecyclerView)
### 들어가기 앞서
전 프로젝트에서 Room을 겪어 보았더니 LiveData,ViewModel까지 이용해야하는 것을 알았다. 이번 프로젝트에서 Room+ViewModel+LiveData를 이용하여 Mvvm에 맞추어 개발 해보자

***
### :wrench: 기능설명

+ MVVM 의 패턴으로 프로젝트를 설계한다.
+ Room을 이용하여 저장하고 데이터를 삭제해준다.
+ RecyclerView를 이용하여 데이터를 리스트형태로 보여준다.
+ LiveData를 사용하여 Room의 정보를 관찰한다.
+ Viewmodel을 통해 중간에서 전달을 해줘 Room과 View를 독립적으로 만들어준다.

### :lollipop: 완성 화면

<img src = "https://user-images.githubusercontent.com/48902047/132084067-6bdc6ea3-c3c0-4f4d-97a7-3a243cff2d84.jpg" width="20%" height="20%"> <img src = "https://user-images.githubusercontent.com/48902047/132084102-25a938ae-9a14-48b5-9b9d-bfea3f02c16c.jpg" width="20%" height="20%"> <img src = "https://user-images.githubusercontent.com/48902047/132084096-ed9e34dd-39fe-4c07-a51c-3bc53d55d2a3.jpg" width="20%" height="20%"> <img src = "https://user-images.githubusercontent.com/48902047/132084110-cd14b74b-e2c8-4bf9-b6e8-87ac3768ad46.jpg" width="20%" height="20%">

***

### Dependency 추가

```Kotlin
plugins {
    id 'com.android.application'
    id 'kotlin-android'
    id 'kotlin-android-extensions'
    id 'kotlin-kapt'
}

android {
    compileSdkVersion 30
    buildToolsVersion "30.0.3"

    defaultConfig {
        applicationId "com.example.mvvmexample"
        minSdkVersion 23
        targetSdkVersion 30
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = '1.8'
    }
}

dependencies {

    implementation "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
    implementation 'androidx.core:core-ktx:1.6.0'
    implementation 'androidx.appcompat:appcompat:1.3.1'
    implementation 'com.google.android.material:material:1.4.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.0'
    testImplementation 'junit:junit:4.+'
    androidTestImplementation 'androidx.test.ext:junit:1.1.3'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.4.0'

    def room_version = "2.2.6"

    implementation "androidx.room:room-runtime:$room_version"
    kapt "androidx.room:room-compiler:$room_version"

    // optional - Kotlin Extensions and Coroutines support for Room
    implementation "androidx.room:room-ktx:$room_version"

    // optional - RxJava support for Room
    implementation "androidx.room:room-rxjava2:$room_version"

    // optional - Guava support for Room, including Optional and ListenableFuture
    implementation "androidx.room:room-guava:$room_version"

    // Test helpers
    testImplementation "androidx.room:room-testing:$room_version"

    // livedata
    implementation 'android.arch.lifecycle:extensions:1.1.1'
    kapt 'android.arch.lifecycle:compiler:1.1.1'

    // recyclerview, cardview
    implementation "androidx.recyclerview:recyclerview:1.0.0"
    implementation "androidx.cardview:cardview:1.0.0"
}
```

### Room 생성
연락처 리스트를 만들 것이므로, 개개인의 연락처를 저장할 클래스를 Entity로 사용하기로 한다.

Contact라는 data class를 만들고 상단에 @Entity 속성을 주어 Entity를 만들었다.

기본키가 되는 id는 @PrimaryKey로 지정하고, null일 경우엔 자동으로 생성되도록 (autoGenerate = true) 속성을 주었다.

Entity에서 테이블 이름을 작성하는 부분인 (tableName = "contact") 부분은 원래 Contact -> contact로 자동으로 변한다. 근데 명시해주는게 나아서 명시해주었다. 아마 db에 저장할때 약속은 spring에서 JPA 처럼 하는거 보니 다 비슷한가보다.


```Kotlin
@Entity(tableName = "contact")
data class Contact(
    @PrimaryKey(autoGenerate = true)
    var id: Long?,

    @ColumnInfo(name = "name")
    var name: String,

    @ColumnInfo(name = "number")
    var number: String,

    @ColumnInfo(name = "initial")
    var initial: Char
) {
    constructor() : this(null, "", "", '\u0000')
}
```

Entity를 만들었으면 SQL을 작성하기 위한 DAO 인터페이스를 만들어준다.

@Query, @Insert, @Update, @Delete 등의 어노테이션을 제공한다. 또한 Insert와 Update에서는 onConflict 속성을 지정할 수 있다. 중복된 데이터의 경우 어떻게 처리할 것인지에 대한 처리를 지정할 수 있다.

주목할 것은 전체 연락처 리스트를 반환하는 getAll 함수를 만들 때 LiveData 를 반환해준다는 점이다. 기존의 익숙한 List<Contact> 형식에 LiveData<>를 감싸주는 방식으로 만들 수 있다. 이렇게 해주면 LiveData가 observe 할 수 있다. 또한 ASC 정렬로 이름순으로 정렬하게 저장하였다.

onConflict = OnConflictStrategy.REPLACE 는 "Insert 할때 PrimaryKey가 겹치는 것이 있으면 덮어 쓴다는 의미이다."
```Kotlin
@Dao
interface ContactDao {

    @Query("SELECT * FROM contact ORDER BY name ASC")
    fun getAll(): LiveData<List<Contact>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    fun insert(contact: Contact)

    @Delete
    fun delete(contact: Contact)

}
```
다음으로 만들 것은 실질적인 데이터베이스 인스턴스를 생성할 Database 클래스이다. RoomDatabase 클래스를 상속하는 추상 클래스로 생성한다.

클래스 이름 위에 @Database 어노테이션을 이용해 entity를 정의하고 SQLite 버전을 지정한다. 또한 데이터베이스 인스턴스를 싱글톤으로 사용하기 위해, companion object 에 만들어주었다.

getInstance 함수는 여러 스레드가 접근하지 못하도록 synchronized (동기화)로 설정한다. 여기서 실질적으로 Room.databaseBuilder 로 인스턴스를 생성하고, fallbackToDestructiveMigration 을 통해 데이터베이스가 갱신될 때 기존의 테이블을 버리고 새로 사용하도록 설정했다.
  
이렇게 만들어지는 DB 인스턴스는 Repository에서 호출하여 사용할 것이다.

 ```Kotlin
@Database(entities = [Contact::class], version = 1)
abstract class ContactDatabase: RoomDatabase() {

    abstract fun contactDao(): ContactDao

    companion object {
        private var INSTANCE: ContactDatabase? = null

        fun getInstance(context: Context): ContactDatabase? {
            if (INSTANCE == null) {
                synchronized(ContactDatabase::class) {
                    INSTANCE = Room.databaseBuilder(context.applicationContext,
                         ContactDatabase::class.java, "contact")
                        .fallbackToDestructiveMigration()
                        .build()
                }
            }
            return INSTANCE
        }
    }

}
``` 

### Repository 생성

사실 Repository에서 크게 정의하는 부분은 없다. 우선 Database, Dao, contacts를 각각 초기화해준다.

그리고 ViewModel에서 DB에 접근을 요청할 때 수행할 함수를 만들어둔다.

룸 프로젝트와 마찬가지로 별도의 스레드에서 Room의 데이터에 접근해야 한다.
  
 ```Kotlin
class ContactRepository(application: Application) {

    private val contactDatabase = ContactDatabase.getInstance(application)!!
    private val contactDao: ContactDao = contactDatabase.contactDao()
    private val contacts: LiveData<List<Contact>> = contactDao.getAll()

    fun getAll(): LiveData<List<Contact>> {
        return contacts
    }

    fun insert(contact: Contact) {
        try {
            val thread = Thread(Runnable {
                contactDao.insert(contact) })
            thread.start()
        } catch (e: Exception) { }
    }

    fun delete(contact: Contact) {
        try {
            val thread = Thread(Runnable {
                contactDao.delete(contact)
            })
            thread.start()
        } catch (e: Exception) { }
    }

}
```
  
### ViewModel 생성
안드로이드 뷰모델 AndroidViewModel을 extend 받는 ContactViewModel을 만들어준다.
  
AndroidViewModel 에서는 Application을 파라미터로 사용한다.
  
Repository를 통해서 Room 데이터베이스의 인스턴스를 만들 때에는 context가 필요하다.
  
하지만, 만약 ViewModel이 액티비티의 context를 쓰게 되면, 액티비티가 destroy 된 경우에는 메모리 릭이 발생할 수 있다. 따라서 Application Context를 사용하기 위해 Applicaion을 인자로 받는다.

DB를 제어할 함수는 Repository에 있는 함수를 이용해 설정해준다.
 ```Kotlin
class ContactViewModel(application: Application) : AndroidViewModel(application) {

    private val repository = ContactRepository(application)
    private val contacts = repository.getAll()

    fun getAll(): LiveData<List<Contact>> {
        return this.contacts
    }

    fun insert(contact: Contact) {
        repository.insert(contact)
    }

    fun delete(contact: Contact) {
        repository.delete(contact)
    }
}
```
  
### View 생성
  
먼저 MainActiviy 부터 설계하겠다.

 ```Kotlin
class MainActivity : AppCompatActivity() {

    private lateinit var contactViewModel: ContactViewModel

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // Set contactItemClick & contactItemLongClick lambda
        val adapter = ContactAdapter({ contact ->
            val intent = Intent(this, AddActivity::class.java)
            intent.putExtra(AddActivity.EXTRA_CONTACT_NAME, contact.name)
            intent.putExtra(AddActivity.EXTRA_CONTACT_NUMBER, contact.number)
            intent.putExtra(AddActivity.EXTRA_CONTACT_ID, contact.id)
            startActivity(intent)
        }, { contact ->
            deleteDialog(contact)
        })

        val lm = LinearLayoutManager(this)
        main_recycleview.adapter = adapter
        main_recycleview.layoutManager = lm
        main_recycleview.setHasFixedSize(true)

        contactViewModel = ViewModelProviders.of(this).get(ContactViewModel::class.java)
        contactViewModel.getAll().observe(this, Observer<List<Contact>> { contacts ->
                adapter.setContacts(contacts!!)
            })

        main_button.setOnClickListener {
            val intent = Intent(this, AddActivity::class.java)
            startActivity(intent)
        }
    }

    private fun deleteDialog(contact: Contact) {
        val builder = AlertDialog.Builder(this)
        builder.setMessage("Delete selected contact?")
            .setNegativeButton("NO") { _, _ -> }
            .setPositiveButton("YES") { _, _ ->
                contactViewModel.delete(contact)
            }
        builder.show()
    }
}
```
  
여기서 ContactViewModel 인스턴스를 만들고, 이를 관찰하는 역할이다.

뷰모델 객체는 직접적으로 초기화 해주는 것이 아니라, 안드로이드 시스템을 통해 생성해준다. 시스템에서는 만약 이미 생성된 ViewModel 인스턴스가 있다면 이를 반환할 것이므로 메모리 낭비를 줄여준다. 따라서 ViewModelProviders를 이용해 get 해준다.
 
 ```Kotlin
 /* MainActivity.kt*/
  
        contactViewModel = ViewModelProviders.of(this).get(ContactViewModel::class.java)
        contactViewModel.getAll().observe(this, Observer<List<Contact>> { contacts ->
                // Update UI
            })
```
Observere를 만들어서 뷰모델이 어느 액티비티/프래그먼트의 생명주기를 관찰할 것인지 정한다. 이 액티비티가 파괴되면 시점에 시스템에서 뷰모델도 자동으로 파괴할 것이다.
옵저버는 아래와 같이 생겼는데 onChanged 메소드를 가지고 있다. 즉, 관찰하고 있던 LiveData가 변하면 무엇을 할 것인지 액션을 지정할 수 있다. 이후 액티비티/프래그먼트가 활성화되어 있다면 View에서 LiveData를 관찰하여 자동으로 변경 사항을 파악하고 이를 수행한다. adapter.setContacts 때 해주었다.(Recyclerview 에서 데이터가 변경될때마다 )

RecyclerView 연결 방법은 건너 뛰고 MainActiviy에 연결한다.
  
다음은 AddActivity 설계이다.
  
 ```Kotlin
class AddActivity : AppCompatActivity() {

    private lateinit var contactViewModel: ContactViewModel
    private var id: Long? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_add)

        contactViewModel = ViewModelProviders.of(this).get(ContactViewModel::class.java)

        // intent null check & get extras
        if (intent != null && intent.hasExtra(EXTRA_CONTACT_NAME) && intent.hasExtra(EXTRA_CONTACT_NUMBER)
            && intent.hasExtra(EXTRA_CONTACT_ID)) {
            add_edittext_name.setText(intent.getStringExtra(EXTRA_CONTACT_NAME))
            add_edittext_number.setText(intent.getStringExtra(EXTRA_CONTACT_NUMBER))
            id = intent.getLongExtra(EXTRA_CONTACT_ID, -1)
        }

        add_button.setOnClickListener {
            val name = add_edittext_name.text.toString().trim()
            val number = add_edittext_number.text.toString()

            if (name.isEmpty() || number.isEmpty()) {
                Toast.makeText(this, "Please enter name and number.", Toast.LENGTH_SHORT).show()
            } else {
                val initial = name[0].toUpperCase()
                val contact = Contact(id, name, number, initial)
                contactViewModel.insert(contact)
                finish()
            }
        }
    }

    companion object {
        const val EXTRA_CONTACT_NAME = "EXTRA_CONTACT_NAME"
        const val EXTRA_CONTACT_NUMBER = "EXTRA_CONTACT_NUMBER"
        const val EXTRA_CONTACT_ID = "EXTRA_CONTACT_ID"
    }
}
``` 
1. intent extra로 사용할 상수를 만든다. (companion object) 
2. ViewModel 객체를 만든다.
3. 만약 intent가 null이 아니고, extra에 주소록 정보가 모두 들어있다면 EditText와 id값을 지정해준다. MainActivity에서 ADD 버튼을 눌렀을 때에는 신규 추가이므로 인텐트가 없고, RecyclerView item 을 눌렀을 때에는 편집을 할 때에는 해당하는 정보를 불러오기 위해 인텐트 값을 불러올 것이다.
4. 하단의 DONE 버튼을 통해 EditText의 null 체크를 한 후, ViewModel을 통해 insert 해주고, MainActivity로 돌아간다.
