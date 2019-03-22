# Network Bound Resource

This abstract class provide an easy interface to fetch resource from both the database and the network.
It works well with the [Android architecture component](https://developer.android.com/topic/libraries/architecture). Most of the code is taken from [this repo](https://github.com/googlesamples/android-architecture-components/tree/master/GithubBrowserSample).

This is just a demo to illustrate the usage of this concept.

### Usage

This class is intended to be called in a `Repository` to retrieve data for a `ViewModel` as described [here](https://developer.android.com/jetpack/docs/guide#overview).
I included in the repo other classes to tie everything together:

* __AppExecutors__ allows the different tasks to run on differents threads
* __ApiResponse__ are the classes for the differents kind of HTTP response
* __Resource__ allows to wrap the differents states of the request (Loading, Success, Error)


#### Example

Let's create a _Todo list_ app to illustrate:

_Task_
```kotlin
@Entity()
data class Task(
    @PrimaryKey
    val id: String,
    val name: String,
    val status: String,
) {}
```

_TaskDao_
```kotlin
@Dao
abstract class TaskDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    abstract fun insert(vararg tasks: Task)

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    abstract fun insertMany(tasks: List<Task>)
    
    @Query("SELECT * FROM Task WHERE status IS :status")
    abstract fun loadTasks(status: String): LiveData<List<Task>>
    
    @Query("SELECT * FROM Task WHERE id == :id")
    abstract fun loadTask(id: String): LiveData<Task>

}
```

_HttpService_
```kotlin
interface HttpService {
    @GET("/tasks/todo")
    fun getTodos(): LiveData<ApiResponse<List<Task>>>
}
```

_TaskRepository_
```kotlin
@Singleton
class TaskRepository @Inject constructor(
    private val appExecutors: AppExecutors,
    private val taskDao: TaskDao,
    private val httpService: HttpService
) {
    fun loadTodos(): LiveData<Resource<List<Task>>> {
        return object : NetworkBoundResource<List<Task>, List<Task>>(appExecutors) {
            override fun saveCallResult(items: List<Task>) {
                taskDao.insertMany(items)
            }

            override fun shouldFetch(data: List<Task>?) = data == null

            override fun loadFromDb(): LiveData<List<Task>> = taskDao.loadTasks("todo")

            override fun createCall() = httpService.getTodos()
        }
    }
}
```

Now in your _ViewModel_:
```kotlin
class TodosViewModel @Inject constructor(
    taskRepository: TaskRepository
) : ViewModel() {
    val todos: LiveDate<Resource<List<Task>>> = taskRepository.loadTodos()
}
```

You should be able to `subscribe` to your `todos` and update your UI when the value change!
