إليك مخطط معماري عملي ومختصر لبناء تطبيق متابعة المصروفات اليومية باستخدام **Jetpack Compose** و **Room** بالتصاميم الحديثة (Unidirectional Data Flow & Clean Architecture).

---

### 1. بنية المشروع (Project Structure)

```text
com.example.expensetracker/
├── data/
│   ├── local/          # Room DB, DAOs, Entities
│   └── repository/     # Repository Implementation
├── domain/
│   ├── model/          # Business Models
│   └── repository/     # Repository Interfaces
└── ui/
    ├── screens/        # Compose Screens & ViewModels
    ├── components/     # Reusable UI Components
    └── theme/          # Material 3 Theme
```

---

### 2. طبقة البيانات (Data Layer - Room)

#### الكيانات (Entities)
```kotlin
// Entity: Transaction
@Entity(tableName = "transactions")
data class TransactionEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val title: String,
    val amount: Double,
    val category: String,
    val date: Long, // System.currentTimeMillis()
    val type: TransactionType // EXPENSE, INCOME
)

enum class TransactionType { EXPENSE, INCOME }
```

#### واجهة التحكم (DAO)
```kotlin
@Dao
interface TransactionDao {
    @Query("SELECT * FROM transactions ORDER BY date DESC")
    fun getAllTransactions(): Flow<List<TransactionEntity>>

    @Query("SELECT SUM(amount) FROM transactions WHERE type = :type")
    fun getTotalByTransactionType(type: TransactionType): Flow<Double?>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertTransaction(transaction: TransactionEntity)

    @Delete
    suspend fun deleteTransaction(transaction: TransactionEntity)
}
```

---

### 3. المستودع وإدارة البيانات (Repository & State Management)

#### المستودع (Repository Interface & Implementation)
* **`ExpenseRepository`**: يوفر البيانات كـ `Flow` لضمان التحديث التلقائي للواجهة (Reactivity).

#### حالة الواجهة (UI State)
```kotlin
data class HomeUiState(
    val transactions: List<TransactionEntity> = emptyList(),
    val totalBalance: Double = 0.0,
    val totalExpense: Double = 0.0,
    val totalIncome: Double = 0.0,
    val isLoading: Boolean = false
)
```

#### نموذج العرض (ViewModel)
```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val repository: ExpenseRepository
) : ViewModel() {

    val uiState: StateFlow<HomeUiState> = combine(
        repository.getAllTransactions(),
        repository.getTotalBalance()
    ) { transactions, balance ->
        HomeUiState(
            transactions = transactions,
            totalBalance = balance,
            isLoading = false
        )
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = HomeUiState(isLoading = true)
    )

    fun addTransaction(title: String, amount: Double, category: String, type: TransactionType) {
        viewModelScope.launch {
            repository.insert(TransactionEntity(title = title, amount = amount, category = category, date = System.currentTimeMillis(), type = type))
        }
    }
    
    fun deleteTransaction(transaction: TransactionEntity) {
        viewModelScope.launch { repository.delete(transaction) }
    }
}
```

---

### 4. الشاشات والواجهات (UI Screens)

1. **شاشة الرئيسية (Home Screen):**
   * **ملخص مالي (Card):** يعرض إجمالي الرصيد، الدخل، والمصروفات.
   * **قائمة المعاملات (LazyColumn):** عرض المصروفات اليومية مع إمكانية السحب للحذف (Swipe-to-Dismiss).
   * **زر إضافة (FAB):** يفتح شاشة/نافذة إضافة مصروف جديد.

2. **شاشة/نافذة الإضافة (Add/Edit Transaction Sheet):**
   * حقول أدخال: العنوان (TextField)، المبلغ (Number Keyboard).
   * اختيار الفئة (FilterChips): (أكل، مواصلات، تسوق، فواتير...).
   * تحديد النوع: (مصروف / دخل) باستخدام `SingleChoiceSegmentedButtonRow`.

3. **شاشة الإحصائيات (Analytics Screen - اختياري):**
   * عرض نسبة المصروفات حسب الفئة باستخدام رسم بياني بسيط (Pie Chart / Progress Bars).

---

### 5. الأذونات (Permissions)

* **لا تتطلب هذه النسخة الأساسية أي أذونات خاصة** (حيث يتم حفظ البيانات محلياً داخل المساحة الخاصة بالتطبيق عبر Room).
* *ملاحظة:* في حال إضافة ميزة "تصدير التقرير كـ PDF/Excel" في المستقبل، ستحتاج إلى إذن إدارة الملفات حسب إصدار Android (مثل `WRITE_EXTERNAL_STORAGE` للإصدارات القديمة أو استخدام `Storage Access Framework` للإصدارات الحديثة).

---

### 6. التبعيات الأساسية (Dependencies)

أضف المكتبات التالية في ملف `build.gradle.kts`:

```kotlin
dependencies {
    // Room Database
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    ksp("androidx.room:room-compiler:2.6.1")

    // Jetpack Compose & ViewModel
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.7.0")

    // Hilt (Dependency Injection)
    implementation("com.google.dagger:hilt-android:2.50")
    ksp("com.google.dagger:hilt-compiler:2.50")
}
``` 