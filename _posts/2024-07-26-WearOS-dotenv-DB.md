---
title: Wear OS 에서 dotenv 파일 및 내부 DB 활용하기
categories:
- Wear OS
- dotenv
- DB
feature_text: "Wear OS 애플리케이션에 보안성과 안정성 보장!"
feature_image: "https://picsum.photos/2560/600?image=872"
author: NurungjiBurger
---
### 들어가며

---

![WearOS](/assets/images/2024-07-26-WearOS-dotenv-DB/WearOS.png)

지난번에는 Wear OS에서 MQTT를 활용하는 방법에 대해서 알아보았습니다. 이번에는 Wear OS에서 프로그램의 **보안성**과 **안정성**을 높혀보고자 했습니다.

# **보안성 고려하기**

---

**dotenv** 즉, “.env” 파일은 환경 변수를 정의하는 데 사용되는 파일로 주로 애플리케이션의 설정을 외부에서 주입하기 위해 사용되며, 개발 및 배포 환경에서 설정을 관리하기 위해 널리 사용됩니다.

보통 애플리케이션의 루트 디렉터리에 위치하며, 각 줄에 **키-값 쌍**으로 환경 변수를 정의합니다. 중요한 정보를 코드에 직접 포함하지 않고 파일로 저장하고 관리하기 때문에 보안 관점에서도 사용하기 좋은 방법입니다.

기본적으로 안드로이드는 우리가 사용하는 PC와는 다른 파일시스템 체계를 사용합니다. 저희가 안드로이드 스튜디오라는 PC 프로그램에서 개발을 진행하고 있지만 에뮬레이터상으로도 그렇고 실제 장치 내부는 완전히 분리된 다른 공간으로 취급하여 개발해야합니다.

우리가 생각했을때 `C:user\android\{projectname}\data\main\.env`의 형태로 접근하면 될 것이라고 생각하겠지만 이는 아예 접근조차 허용하지 않습니다.

이에 저는 2가지 방법을 시도해 보았습니다.

## 1. adb shell

---

안드로이드 스튜디오의 왼쪽 하단을 보면 로그캣, 터미널, 빌드 등 다양한 메뉴가 있습니다. 이 중 터미널에서 adb shell을 활용해 linux 파일 시스템에 접근하는 방식처럼 안드로이드의 파일 시스템에 접근하여 echo 를 활용해 파일을 직접 만들어 보려고 했습니다.

결론적으로 echo를 활용한 방법은 **permission denied** 당했습니다. 안드로이드 디바이스 파일 시스템에서 adb shell 명령어로 접근할 수 있는 기본 권한은 매우 제한적이며 ‘/system’과 같은 시스템 파티션은 루트 권한을 통해 접근 할 수 있습니다. 하지만 sudo 명령어를 동반해 사용해봤지만 이 역시 거부 당했습니다.

안드로이드는 SELinux 보안 정책을 사용해 파일 시스템에 대한 접근을 제한하고 있는데 이는 특정 디렉터리나 파일에 대한 접근을 강하게 제한하고 있습니다. 또한 AOSP ( Android Open Source Project ) 에뮬레이터는 기본적으로 루트 권한을 제공하지 않기 때문에 sudo를 동반한 명령 또한 실패할 수 밖에 없었습니다.

## 2. Device Explorer (최종 선택)

---

안드로이드 스튜디오는 **Device Explorer**를 제공합니다. 이는 에뮬레이터의 파일 시스템을 PC의 파일 탐색기 처럼 눈으로 볼 수 있도록 제공하는 도구입니다.

user\data\data\com.example.{project} 등의 경로를 해당 탐색기에서 찾아 dotenv 파일을 직접 생성해서 사용하려고 하였습니다.

하지만, 파일 생성까지는 문제가 없었지만 데이터를 읽고 쓰는 모든 권한이 있는 상태였지만 해당 파일에 데이터를 쓰고 저장한 뒤 다시 열어보았을 때는 데이터가 항상 비어있었고 실제로 코드상에서 파일에 접근, 데이터를 로드 했을때 아무런 내용을 불러오지 못하고 있었습니다.

파일 시스템이 변경된 후 동기화가 되지 않아서 저장되지 않은 것은 아닌지, 사실 파일을 변경할 권한이 없던 것은 아닌지 등을 검토 했지만 모두 정상이었기 때문에 에뮬레이터 상의 버그나 문제로 판단했습니다.

> Drag & Drop 방식으로 해결
> 

마지막으로 외부에서 생성한 dotenv 파일을 Device Explorer에 **drag & drop** 방식으로 옮겨보는 것을 시도했습니다.

실제로 파일은 정상적으로 파일 시스템에 제대로 들어간 것으로 보였고 열어봤을 때 내용도 제대로 들어가 있는 것을 확인할 수 있었습니다. 위 과정을 통해 코드 상으로 파일을 생성하고 직접 write 하는 방법도 고려해볼 수 있었다고 생각하게 되었으나 시도해보지는 않았습니다.

# dotenv 사용하기

---

안드로이드 스튜디오의 코드상에서 dotenv 파일을 사용하기 위한 방법을 소개하도록 하겠습니다.

### build.gradle.kts

```kotlin
    implementation(libs.cdimascio.dotenv.kotlin)
    implementation(libs.dotenv.kotlin.v640)
```

build.gradle 파일에서 해당 라이브러리를 사용하기 위해 위와같은 내용을 추가해주어야 합니다.

dotenv를 지원하는 라이브러리 의존성을 추가해주었습니다.

### 프로젝트 클래스

```kotlin
import io.github.cdimascio.dotenv.dotenv

		// dotenv 파일 불러오기
    func loadEnv {
        try {
            val dotenv = dotenv {
                // 앱 내부
                // 여기서 경로는 /data/user/0/project_name/files/.env
                directory = context.filesDir.absolutePath
                filename = ".env"
            }

            // 찾았으면 할당
            val MQTT_BROKER_ID = dotenv["MQTT_BROKER_IP"]
            
            // 해당 IP로 연결
            mqttClient = MqttClient(MQTT_BROKER_ID, MqttClient.generateClientId(), MemoryPersistence())
        } catch (e: Exception) {
            // 에러처리
        }
    }
```

프로젝트의 클래스에서 해당 파일을 사용하기 위한 예시코드 입니다.

dotenv를 사용하기위해 임포트하고 코드상에서 dotenv파일을 불러와야합니다. 저는 mqtt broker ip를 보호해야하는 정보로 생각하고 코드를 작성하였습니다.

directory로 사용된 context.filesDir.absolutePath는 주석에 쓰인 것처럼 해당 프로젝트의 절대 경로를 표현합니다. 안드로이드 스튜디오에서는 기본적으로 context.fileDir.absolutePath, context.assets, context.resources 등을 제공하고 있는데 이는 안드로이드 시스템에서의 특정 경로들을 의미하는 것으로 본인에게 필요한 것을 적절하게 이용하시면 됩니다.

# 안정성 고려하기

---

저희는 예기치 못한 상황 발생으로 애플리케이션이나 장치가 셧다운된다면 기존에 처리하고 있던 루틴이 완료되지 못한채로 계속해서 기다리게되는 교착상태에 빠질 수 있다고 예상했습니다.

따라서 Wear OS가 자체적으로 현재 작업중인 물류 데이터에 대해서 저장해두고 있을 필요성을 느끼게 되었고, 위 문제를 해결하기 위해 이용할 수 있는 3가지 방법을 찾았습니다.

## SharedPreference

---

key-value 쌍으로 데이터를 저장하고 불러오는 방식으로 사용자의 로그인 토큰 등을 저장하는데 사용하는 방식입니다.

암호화 옵션이 내장되어 있어 보안성이 뛰어나지만 데이터 크기가 제한되어 있기에 물류데이터처럼 많은 양의 데이터를 저장하기에는 적절하지 않을 수 있습니다.

## File Storage

---

파일 형식으로 데이터를 저장할 수 있어 데이터 구조에 제한이 없으며 대량의 데이터를 저장하는게 가능한 방식입니다. 

너무 많은 데이터를 다룰 때는 성능 저하가 있을 수 있으며 상태 변화에 맞추어 데이터의 업데이트가 빈번하게 일어나는 경우에는 알맞지 않은 방법입니다.

## Room Database (최종선택)

---

sql 형식으로 데이터를 구조화해서 저장할 수 있으며 대량의 데이터를 저장하고 관리하기에 적합한 방식입니다. 

안드로이드 애플리케이션에서 SQLite 데이터베이스를 쉽게 사용할 수 있도록 라이브러리를 제공해주고 있고 SQLite의 복잡한 작업을 추상화해 간단하고 안전하게 데이터베이스 작업을 수행할 수 있도록 도와줍니다.

저희의 물류 시스템은 물류 하나의 처리가 완료될 때마다 해당 물류의 상태를 업데이트 하는 일이 빈번하게 일어날 것이고 데이터 양이 많기 때문에 Room Database를 활용하는 것이 적합하다고 판단하였습니다.

# Room DB 사용하기

---

안드로이드 스튜디오를 사용한 Wear OS를 개발하고 있었기 때문에 Room Database 또한 라이브러리로 제공받아 사용할 수 있었습니다. 다음은 사용 과정에 대한 설명입니다.

### build.gradle.kts

```kotlin
    // plugin
    id("kotlin-kapt")
    // dependencies
    implementation(libs.androidx.room.runtime)
    implementation(libs.room.ktx)
    kapt(libs.androidx.room.compiler)
```

build.gradle 파일에서 해당 라이브러리를 사용하기 위해 위와같은 내용을 추가해주어야 합니다.

Room 라이브러리는 컴파일 타임에 코드 생성을 위해 Annotation Prcessing을 사용하는데 코틀린에서 이를 사용하기 위해서는 플러그인부터 설정해주어야합니다.

이후 의존성 추가를 통해 room 라이브러리와 kapt( Kotlin Annotation Processing Tool )를 사용할 수 있도록 정의해줍니다.

### Data Entity

```kotlin
@Entity(tableName = "logistics")
data class Logistics(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val name: String,
    val quantity: Int,
    val status: String
)
```

SQLite 데이터베이스를 사용하기 위해서 필요한 테이블을 정의해야 합니다.

autoGenerate는 데이터베이스에서 autoincrement를 의미하며 데이터가 들어올때마다 하나씩 증가하는 ID를 이용할 수 있도록 했고 이를 기본키로 사용하였습니다.

그 외에 데이터들은 프로젝트에서 필요한 내용들로 구성되어있습니다.

### DAO Interface

```kotlin
@Dao
interface LogisticsDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(logistics: List<Logistics>)

    @Query("DELETE FROM logistics")
    suspend fun deleteAll()

    @Query("SELECT * FROM logistics")
    suspend fun getAll(): List<Logistics>
}
```

다음으로 Room 라이브러리에서 데이터베이스 작업을 어떻게 처리할지 인터페이스를 정의해주어야 합니다.

보통 이는 DAO ( Data Access Object ) 인터페이스에서 정의하고 처리하게 됩니다. DAO는 데이터베이스에 대한 CRUD 작업을 정의하는데 사용되며 지정된 함수에 대해서 어노테이션으로 지정된 쿼리가 실행되도록 구성되어 있습니다.

특히나 데이터베이스와 관련된 작업은 비동기적으로 처리되어야하기 때문에 메인 스레드에서 실행되지 않도록 따로 처리 할 수 있어야하며 메인 스레드를 차단하지 않도록 설계되어야 합니다. 

또한, 코드상에서 이를 함수 앞에 suspend 키워드로 등록하여 코루틴과 같은 비동기 작업을 위한 작업임을 한눈에 알 수 있고 처리하도록 설정해주어야 합니다.

마지막으로 데이터베이스는 아래와 같은 형태로 정의되어지며 우리가 사용할 수 있도록 준비를 마치게 됩니다.

```kotlin
@Database(entities = [Logistics::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun logisticsDao(): LogisticsDao
}
```

### 프로젝트 클래스

```kotlin
    // Room Database 불러오기, 없다면 생성
    private var database: MullyuRoomDataBase = Room.databaseBuilder(
        application,
        MullyuRoomDataBase::class.java, "mullyuDatabase"
    ).build()

    // Confirm 버튼을 누르면 해당 물류에 대한 처리가 완료되었음을 의미
    fun ConfirmMullyuData() {
        val currentData = _mullyuData.value ?: return
        // 처리 완료 표시 및 DB 업데이트
        if (!currentData.isProcess) {
            viewModelScope.launch(Dispatchers.IO) {
                currentData.isProcess = true
                _processCount.value += 1
                // DB의 특정 물품에 대해 업데이트 적용
                database.mullyuLogisticsDao().updateIsProcess(currentData.name)
            }
        }
        displayNextMullyuData()
    }

    // 데이터리스트 업데이트
    fun updateDataList(newDataList: List<MullyuLogistics>) {
        viewModelScope.launch(Dispatchers.IO) {
            // 기존 DB 삭제
            database.mullyuLogisticsDao().delete()
            val lastInsertedId = database.mullyuLogisticsDao().getLastInsertedId() ?: 0
            // ID가 너무 커지는 것을 방지하기 위해 ID가 너무 커지지 않도록 한번씩 리셋
            if (lastInsertedId >= 1000000) {
                database.mullyuLogisticsDao().deleteAll()
                database.clearAllTables()
            }
            // 데이터를 다 썼을 때는 다시 No Data 표시
            if (newDataList.size == 0) {
                _dataList.value = emptyList()
                _mullyuData.value = null
            }
            else {
                // 새로운 DB 삽입
                database.mullyuLogisticsDao().insertAll(newDataList)
                _dataList.value = newDataList
                _mullyuData.value = newDataList.getOrNull(0)
            }
            // 처리 완료된 데이터의 수 표시
            _processCount.value = _dataList.value.count { it.isProcess }
        }
    }

    // 데이터베이스에서 모든 데이터를 가져오는 suspend 함수
    suspend fun getAllDataFromDatabase(): List<MullyuLogistics> {
        return withContext(Dispatchers.IO) {
            try {
                database.mullyuLogisticsDao().getAll()
            } catch (e: Exception) {
                println("데이터베이스 읽기 오류: ${e.message}")
                emptyList()
            }
        }
    }

    // 데이터베이스에서 모든 데이터를 가져와서 출력
    fun printAllData() {
        viewModelScope.launch(Dispatchers.IO) {
            val db = getDatabase()
            val dataList = db.mullyuLogisticsDao().getAll()
            println("전체 데이터:")
            dataList.forEach {
                println("번호: ${it.id}, 물류이름: ${it.name}, 물류량: ${it.quantity}, 처리상태: ${it.isProcess}")
            }
        }
    }
```

사용할 준비를 마친 데이터베이스는 위와 같은 코드로 작동하게 됩니다.

주석과 같은 과정으로 작동하게 되는데 주의해야할 점은 메인 스레드에서 직접 호출하지 않도록 주의해야한다는 점입니다.

앞서 말씀드렸듯이 데이터베이스 작업은 비동기적으로 처리되어야하는 느린 작업이기 때문에 코루틴 작업 환경에서 처리되어야 하기 때문입니다.

# 글을 마무리하며

---

개발을 진행하면서 보안성이나 안정성에 대한 고민을 많이 해본 적이 없었는데 위 내용들을 겪으면서 더 좋은 사용자 경험을 위한 프로그램에 대해서 생각해본 시간이었던 것 같습니다. 특히나 다양한 해결 방법들 중에서 현재 프로젝트에 가장 알맞은 방법들을 선택하면서 프로젝트의 효율성에 대해서도 고민할 수 있는 시간이었습니다.