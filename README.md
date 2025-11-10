# doutor-em-casa
Repositório para aplicativo Doutor em casa, projeito de extensão da Estácio
# Projeto: Doutor em Casa (Kotlin) - Código completo
---

## 1) build.gradle (module: app)
```gradle
plugins {
    id 'com.android.application'
    id 'kotlin-android'
    id 'kotlin-kapt'
}

android {
    namespace 'br.com.doutoremcasa'
    compileSdk 34

    defaultConfig {
        applicationId "br.com.doutoremcasa"
        minSdk 21
        targetSdk 34
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_17
        targetCompatibility JavaVersion.VERSION_17
    }
    kotlinOptions {
        jvmTarget = '17'
    }
}

dependencies {
    implementation 'androidx.core:core-ktx:1.10.1'
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.9.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'

    implementation 'androidx.recyclerview:recyclerview:1.3.0'

    // Room
    implementation 'androidx.room:room-runtime:2.6.1'
    kapt 'androidx.room:room-compiler:2.6.1'
    implementation 'androidx.room:room-ktx:2.6.1'

    // Lifecycle
    implementation 'androidx.lifecycle:lifecycle-runtime-ktx:2.6.1'

    // Coroutines
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.1'

    // CardView
    implementation 'androidx.cardview:cardview:1.0.0'
}
```

---

## 2) AndroidManifest.xml (no app/src/main/AndroidManifest.xml)
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="br.com.doutoremcasa">

    <application
        android:allowBackup="true"
        android:label="@string/app_name"
        android:icon="@mipmap/ic_launcher"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:theme="@style/Theme.DoutorEmCasa">

        <activity android:name="br.com.doutoremcasa.ui.AppointmentActivity" />
        <activity android:name="br.com.doutoremcasa.ui.PatientDetailActivity" />
        <activity android:name="br.com.doutoremcasa.ui.PatientListActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

---

## 3) Estrutura de pacotes (em `app/src/main/java/br/com/doutoremcasa`)
- data.local.model
- data.local.dao
- data.local
- repository
- ui
  - adapters
  - dialogs
- util

---

## 4) Código Kotlin

### 4.1 Modelos (Room entities)

`app/src/main/java/br/com/doutoremcasa/data/local/model/Patient.kt`
```kotlin
package br.com.doutoremcasa.data.local.model

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "patients")
data class Patient(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val name: String,
    val birthDate: String? = null,
    val phone: String? = null,
    val address: String? = null
)
```

`app/src/main/java/br/com/doutoremcasa/data/local/model/MedicalRecord.kt`
```kotlin
package br.com.doutoremcasa.data.local.model

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "medical_records")
data class MedicalRecord(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val patientId: Long,
    val date: String,
    val summary: String,
    val doctorNotes: String? = null
)
```

`app/src/main/java/br/com/doutoremcasa/data/local/model/Appointment.kt`
```kotlin
package br.com.doutoremcasa.data.local.model

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "appointments")
data class Appointment(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val patientId: Long,
    val dateTime: String,
    val address: String,
    val notes: String? = null
)
```

---

### 4.2 DAOs

`app/src/main/java/br/com/doutoremcasa/data/local/dao/PatientDao.kt`
```kotlin
package br.com.doutoremcasa.data.local.dao

import androidx.room.*
import br.com.doutoremcasa.data.local.model.Patient

@Dao
interface PatientDao {
    @Query("SELECT * FROM patients ORDER BY name ASC")
    suspend fun getAll(): List<Patient>

    @Query("SELECT * FROM patients WHERE id = :id LIMIT 1")
    suspend fun getById(id: Long): Patient?

    @Insert
    suspend fun insert(patient: Patient): Long

    @Update
    suspend fun update(patient: Patient)

    @Delete
    suspend fun delete(patient: Patient)
}
```

`app/src/main/java/br/com/doutoremcasa/data/local/dao/MedicalRecordDao.kt`
```kotlin
package br.com.doutoremcasa.data.local.dao

import androidx.room.*
import br.com.doutoremcasa.data.local.model.MedicalRecord

@Dao
interface MedicalRecordDao {
    @Query("SELECT * FROM medical_records WHERE patientId = :pId ORDER BY date DESC")
    suspend fun recordsForPatient(pId: Long): List<MedicalRecord>

    @Insert
    suspend fun insert(record: MedicalRecord): Long
}
```

`app/src/main/java/br/com/doutoremcasa/data/local/dao/AppointmentDao.kt`
```kotlin
package br.com.doutoremcasa.data.local.dao

import androidx.room.*
import br.com.doutoremcasa.data.local.model.Appointment

@Dao
interface AppointmentDao {
    @Query("SELECT * FROM appointments ORDER BY dateTime ASC")
    suspend fun getAll(): List<Appointment>

    @Insert
    suspend fun insert(appointment: Appointment): Long

    @Query("SELECT * FROM appointments WHERE patientId = :pId ORDER BY dateTime ASC")
    suspend fun appointmentsForPatient(pId: Long): List<Appointment>
}
```

---

### 4.3 AppDatabase

`app/src/main/java/br/com/doutoremcasa/data/local/AppDatabase.kt`
```kotlin
package br.com.doutoremcasa.data.local

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase
import br.com.doutoremcasa.data.local.dao.AppointmentDao
import br.com.doutoremcasa.data.local.dao.MedicalRecordDao
import br.com.doutoremcasa.data.local.dao.PatientDao
import br.com.doutoremcasa.data.local.model.Appointment
import br.com.doutoremcasa.data.local.model.MedicalRecord
import br.com.doutoremcasa.data.local.model.Patient

@Database(entities = [Patient::class, MedicalRecord::class, Appointment::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun patientDao(): PatientDao
    abstract fun medicalRecordDao(): MedicalRecordDao
    abstract fun appointmentDao(): AppointmentDao

    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase =
            INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "doutor_em_casa_db"
                ).build()
                INSTANCE = instance
                instance
            }
    }
}
```

---

### 4.4 Repository

`app/src/main/java/br/com/doutoremcasa/repository/AppRepository.kt`
```kotlin
package br.com.doutoremcasa.repository

import android.content.Context
import br.com.doutoremcasa.data.local.AppDatabase
import br.com.doutoremcasa.data.local.model.Appointment
import br.com.doutoremcasa.data.local.model.MedicalRecord
import br.com.doutoremcasa.data.local.model.Patient

class AppRepository(context: Context) {
    private val db = AppDatabase.getInstance(context)
    private val patientDao = db.patientDao()
    private val recordDao = db.medicalRecordDao()
    private val appointmentDao = db.appointmentDao()

    suspend fun getAllPatients() = patientDao.getAll()
    suspend fun getPatient(id: Long) = patientDao.getById(id)
    suspend fun addPatient(p: Patient) = patientDao.insert(p)
    suspend fun updatePatient(p: Patient) = patientDao.update(p)
    suspend fun deletePatient(p: Patient) = patientDao.delete(p)

    suspend fun getRecordsFor(patientId: Long) = recordDao.recordsForPatient(patientId)
    suspend fun addRecord(r: MedicalRecord) = recordDao.insert(r)

    suspend fun getAllAppointments() = appointmentDao.getAll()
    suspend fun addAppointment(a: Appointment) = appointmentDao.insert(a)
    suspend fun getAppointmentsFor(patientId: Long) = appointmentDao.appointmentsForPatient(patientId)
}
```

---

### 4.5 Util

`app/src/main/java/br/com/doutoremcasa/util/DateUtils.kt`
```kotlin
package br.com.doutoremcasa.util

import java.text.SimpleDateFormat
import java.util.*

object DateUtils {
    private val sdf = SimpleDateFormat("yyyy-MM-dd HH:mm", Locale.getDefault())
    fun nowString(): String = sdf.format(Date())
}
```

---

### 4.6 UI - Activities

`app/src/main/java/br/com/doutoremcasa/ui/PatientListActivity.kt`
```kotlin
package br.com.doutoremcasa.ui

import android.content.Intent
import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import androidx.recyclerview.widget.LinearLayoutManager
import br.com.doutoremcasa.databinding.ActivityPatientListBinding
import br.com.doutoremcasa.repository.AppRepository
import br.com.doutoremcasa.data.local.model.Patient
import br.com.doutoremcasa.ui.adapters.PatientAdapter
import br.com.doutoremcasa.ui.dialogs.AddPatientDialog
import kotlinx.coroutines.launch
import androidx.lifecycle.lifecycleScope

class PatientListActivity : AppCompatActivity() {
    private lateinit var binding: ActivityPatientListBinding
    private lateinit var repo: AppRepository
    private lateinit var adapter: PatientAdapter

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityPatientListBinding.inflate(layoutInflater)
        setContentView(binding.root)

        repo = AppRepository(applicationContext)

        adapter = PatientAdapter { patient ->
            val i = Intent(this, PatientDetailActivity::class.java)
            i.putExtra("patientId", patient.id)
            startActivity(i)
        }

        binding.recyclerPatients.layoutManager = LinearLayoutManager(this)
        binding.recyclerPatients.adapter = adapter

        binding.fabAddPatient.setOnClickListener {
            AddPatientDialog.show(supportFragmentManager) { name, birth, phone, address ->
                lifecycleScope.launch {
                    repo.addPatient(Patient(name = name, birthDate = birth, phone = phone, address = address))
                    loadPatients()
                }
            }
        }

        loadPatients()
    }

    override fun onResume() {
        super.onResume()
        loadPatients()
    }

    private fun loadPatients() {
        lifecycleScope.launch {
            val list = repo.getAllPatients()
            adapter.submitList(list)
            binding.tvEmpty.visibility = if (list.isEmpty()) android.view.View.VISIBLE else android.view.View.GONE
        }
    }
}
```

`app/src/main/java/br/com/doutoremcasa/ui/PatientDetailActivity.kt`
```kotlin
package br.com.doutoremcasa.ui

import android.os.Bundle
import androidx.appcompat.app.AlertDialog
import androidx.appcompat.app.AppCompatActivity
import androidx.recyclerview.widget.LinearLayoutManager
import br.com.doutoremcasa.databinding.ActivityPatientDetailBinding
import br.com.doutoremcasa.repository.AppRepository
import br.com.doutoremcasa.ui.adapters.RecordAdapter
import br.com.doutoremcasa.ui.dialogs.AddRecordDialog
import kotlinx.coroutines.launch
import androidx.lifecycle.lifecycleScope

class PatientDetailActivity : AppCompatActivity() {
    private lateinit var binding: ActivityPatientDetailBinding
    private lateinit var repo: AppRepository
    private var patientId: Long = 0
    private lateinit var recordAdapter: RecordAdapter

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityPatientDetailBinding.inflate(layoutInflater)
        setContentView(binding.root)

        repo = AppRepository(applicationContext)
        patientId = intent.getLongExtra("patientId", 0)

        recordAdapter = RecordAdapter()
        binding.recyclerRecords.layoutManager = LinearLayoutManager(this)
        binding.recyclerRecords.adapter = recordAdapter

        binding.btnAddRecord.setOnClickListener {
            AddRecordDialog.show(supportFragmentManager, patientId) { date, summary, notes ->
                lifecycleScope.launch {
                    repo.addRecord(br.com.doutoremcasa.data.local.model.MedicalRecord(patientId = patientId, date = date, summary = summary, doctorNotes = notes))
                    loadData()
                }
            }
        }

        binding.btnSchedule.setOnClickListener {
            val i = android.content.Intent(this, AppointmentActivity::class.java)
            i.putExtra("patientId", patientId)
            startActivity(i)
        }

        binding.btnDeletePatient.setOnClickListener {
            lifecycleScope.launch {
                val p = repo.getPatient(patientId)
                if (p != null) {
                    AlertDialog.Builder(this@PatientDetailActivity)
                        .setTitle("Excluir paciente")
                        .setMessage("Deseja realmente excluir ${p.name}?")
                        .setPositiveButton("Sim") { _, _ ->
                            lifecycleScope.launch { repo.deletePatient(p); finish() }
                        }
                        .setNegativeButton("Não", null)
                        .show()
                }
            }
        }

        loadData()
    }

    private fun loadData() {
        lifecycleScope.launch {
            val p = repo.getPatient(patientId)
            if (p != null) {
                binding.tvName.text = p.name
                binding.tvPhone.text = p.phone ?: "-"
                binding.tvAddress.text = p.address ?: "-"
                binding.tvBirth.text = p.birthDate ?: "-"
            }

            val records = repo.getRecordsFor(patientId)
            recordAdapter.submitList(records)
            binding.tvRecordsCount.text = "${records.size} registro(s)"
            binding.tvNoRecords.visibility = if (records.isEmpty()) android.view.View.VISIBLE else android.view.View.GONE
        }
    }
}
```

`app/src/main/java/br/com/doutoremcasa/ui/AppointmentActivity.kt`
```kotlin
package br.com.doutoremcasa.ui

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import br.com.doutoremcasa.databinding.ActivityAppointmentBinding
import br.com.doutoremcasa.repository.AppRepository
import br.com.doutoremcasa.data.local.model.Appointment
import br.com.doutoremcasa.util.DateUtils
import kotlinx.coroutines.launch
import androidx.lifecycle.lifecycleScope

class AppointmentActivity : AppCompatActivity() {
    private lateinit var binding: ActivityAppointmentBinding
    private lateinit var repo: AppRepository
    private var patientId: Long = 0

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityAppointmentBinding.inflate(layoutInflater)
        setContentView(binding.root)

        repo = AppRepository(applicationContext)
        patientId = intent.getLongExtra("patientId", 0)

        // Preencher data/hora com valor atual
        binding.etDateTime.setText(DateUtils.nowString())

        binding.btnSave.setOnClickListener {
            val datetime = binding.etDateTime.text.toString().trim()
            val address = binding.etAddress.text.toString().trim()
            val notes = binding.etNotes.text.toString().trim().ifEmpty { null }

            if (datetime.isEmpty() || address.isEmpty()) {
                android.widget.Toast.makeText(this, "Preencha data/hora e endereço", android.widget.Toast.LENGTH_SHORT).show()
                return@setOnClickListener
            }

            lifecycleScope.launch {
                repo.addAppointment(Appointment(patientId = patientId, dateTime = datetime, address = address, notes = notes))
                finish()
            }
        }
    }
}
```

---

### 4.7 UI - Adapters

`app/src/main/java/br/com/doutoremcasa/ui/adapters/PatientAdapter.kt`
```kotlin
package br.com.doutoremcasa.ui.adapters

import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.DiffUtil
import androidx.recyclerview.widget.ListAdapter
import androidx.recyclerview.widget.RecyclerView
import br.com.doutoremcasa.data.local.model.Patient
import br.com.doutoremcasa.databinding.ItemPatientBinding

class PatientAdapter(private val onClick: (Patient) -> Unit) : ListAdapter<Patient, PatientAdapter.Holder>(DIFF) {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): Holder {
        val binding = ItemPatientBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return Holder(binding)
    }

    override fun onBindViewHolder(holder: Holder, position: Int) {
        val p = getItem(position)
        holder.bind(p)
    }

    inner class Holder(private val b: ItemPatientBinding) : RecyclerView.ViewHolder(b.root) {
        fun bind(patient: Patient) {
            b.tvName.text = patient.name
            b.tvPhone.text = patient.phone ?: ""
            b.root.setOnClickListener { onClick(patient) }
        }
    }

    companion object {
        val DIFF = object : DiffUtil.ItemCallback<Patient>() {
            override fun areItemsTheSame(oldItem: Patient, newItem: Patient) = oldItem.id == newItem.id
            override fun areContentsTheSame(oldItem: Patient, newItem: Patient) = oldItem == newItem
        }
    }
}
```

`app/src/main/java/br/com/doutoremcasa/ui/adapters/RecordAdapter.kt`
```kotlin
package br.com.doutoremcasa.ui.adapters

import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.DiffUtil
import androidx.recyclerview.widget.ListAdapter
import androidx.recyclerview.widget.RecyclerView
import br.com.doutoremcasa.data.local.model.MedicalRecord
import br.com.doutoremcasa.databinding.ItemRecordBinding

class RecordAdapter : ListAdapter<MedicalRecord, RecordAdapter.Holder>(DIFF) {
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): Holder {
        val binding = ItemRecordBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return Holder(binding)
    }

    override fun onBindViewHolder(holder: Holder, position: Int) {
        holder.bind(getItem(position))
    }

    inner class Holder(private val b: ItemRecordBinding) : RecyclerView.ViewHolder(b.root) {
        fun bind(r: MedicalRecord) {
            b.tvDate.text = r.date
            b.tvSummary.text = r.summary
            b.tvNotes.text = r.doctorNotes ?: ""
        }
    }

    companion object {
        val DIFF = object : DiffUtil.ItemCallback<MedicalRecord>() {
            override fun areItemsTheSame(oldItem: MedicalRecord, newItem: MedicalRecord) = oldItem.id == newItem.id
            override fun areContentsTheSame(oldItem: MedicalRecord, newItem: MedicalRecord) = oldItem == newItem
        }
    }
}
```

---

### 4.8 UI - DialogFragments

`app/src/main/java/br/com/doutoremcasa/ui/dialogs/AddPatientDialog.kt`
```kotlin
package br.com.doutoremcasa.ui.dialogs

import android.app.Dialog
import android.os.Bundle
import androidx.appcompat.app.AlertDialog
import androidx.fragment.app.DialogFragment
import br.com.doutoremcasa.databinding.DialogAddPatientBinding

class AddPatientDialog : DialogFragment() {
    private var _binding: DialogAddPatientBinding? = null
    private val binding get() = _binding!!

    companion object {
        fun show(fm: androidx.fragment.app.FragmentManager, onSaved: (name: String, birth: String?, phone: String?, address: String?) -> Unit) {
            val dlg = AddPatientDialog()
            dlg.onSaved = onSaved
            dlg.show(fm, "AddPatientDialog")
        }
    }

    private var onSaved: ((String, String?, String?, String?) -> Unit)? = null

    override fun onCreateDialog(savedInstanceState: Bundle?): Dialog {
        _binding = DialogAddPatientBinding.inflate(layoutInflater)

        val builder = AlertDialog.Builder(requireContext())
            .setTitle("Adicionar paciente")
            .setView(binding.root)
            .setPositiveButton("Salvar") { _, _ ->
                val name = binding.etName.text.toString().trim()
                val birth = binding.etBirth.text.toString().trim().ifEmpty { null }
                val phone = binding.etPhone.text.toString().trim().ifEmpty { null }
                val address = binding.etAddress.text.toString().trim().ifEmpty { null }
                if (name.isNotEmpty()) onSaved?.invoke(name, birth, phone, address)
            }
            .setNegativeButton("Cancelar", null)

        return builder.create()
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

`app/src/main/java/br/com/doutoremcasa/ui/dialogs/AddRecordDialog.kt`
```kotlin
package br.com.doutoremcasa.ui.dialogs

import android.app.Dialog
import android.os.Bundle
import androidx.appcompat.app.AlertDialog
import androidx.fragment.app.DialogFragment
import br.com.doutoremcasa.databinding.DialogAddRecordBinding
import br.com.doutoremcasa.util.DateUtils

class AddRecordDialog : DialogFragment() {
    private var _binding: DialogAddRecordBinding? = null
    private val binding get() = _binding!!

    companion object {
        fun show(fm: androidx.fragment.app.FragmentManager, patientId: Long, onSaved: (date: String, summary: String, notes: String?) -> Unit) {
            val dlg = AddRecordDialog()
            dlg.onSaved = onSaved
            dlg.show(fm, "AddRecordDialog")
        }
    }

    private var onSaved: ((String, String, String?) -> Unit)? = null

    override fun onCreateDialog(savedInstanceState: Bundle?): Dialog {
        _binding = DialogAddRecordBinding.inflate(layoutInflater)
        binding.etDate.setText(DateUtils.nowString())

        val builder = AlertDialog.Builder(requireContext())
            .setTitle("Adicionar prontuário")
            .setView(binding.root)
            .setPositiveButton("Salvar") { _, _ ->
                val date = binding.etDate.text.toString().trim()
                val summary = binding.etSummary.text.toString().trim()
                val notes = binding.etNotes.text.toString().trim().ifEmpty { null }
                if (summary.isNotEmpty()) onSaved?.invoke(date, summary, notes)
            }
            .setNegativeButton("Cancelar", null)

        return builder.create()
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

---

## 5) Layouts XML (em `app/src/main/res/layout`)

### activity_patient_list.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/tvEmpty"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Nenhum paciente cadastrado"
        android:layout_gravity="center"
        android:visibility="gone" />

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerPatients"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:padding="16dp" />

    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:id="@+id/fabAddPatient"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_margin="16dp"
        app:layout_anchorGravity="bottom|end"
        app:srcCompat="@android:drawable/ic_input_add" />

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

### item_patient.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.cardview.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_marginBottom="8dp"
    android:padding="12dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">

        <TextView
            android:id="@+id/tvName"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:textSize="18sp"
            android:textStyle="bold" />

        <TextView
            android:id="@+id/tvPhone"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:textSize="14sp" />

    </LinearLayout>
</androidx.cardview.widget.CardView>
```

### activity_patient_detail.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="16dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">

        <TextView android:id="@+id/tvName" android:layout_width="match_parent" android:layout_height="wrap_content" android:textSize="20sp" android:textStyle="bold" />
        <TextView android:id="@+id/tvPhone" android:layout_width="match_parent" android:layout_height="wrap_content" />
        <TextView android:id="@+id/tvAddress" android:layout_width="match_parent" android:layout_height="wrap_content" />
        <TextView android:id="@+id/tvBirth" android:layout_width="match_parent" android:layout_height="wrap_content" />

        <Button android:id="@+id/btnAddRecord" android:layout_width="match_parent" android:layout_height="wrap_content" android:text="Adicionar prontuário" />
        <Button android:id="@+id/btnSchedule" android:layout_width="match_parent" android:layout_height="wrap_content" android:text="Agendar consulta" />
        <Button android:id="@+id/btnDeletePatient" android:layout_width="match_parent" android:layout_height="wrap_content" android:text="Excluir paciente" />

        <TextView android:id="@+id/tvRecordsCount" android:layout_width="match_parent" android:layout_height="wrap_content" android:layout_marginTop="8dp" />

        <TextView android:id="@+id/tvNoRecords" android:layout_width="match_parent" android:layout_height="wrap_content" android:text="Nenhum prontuário" android:visibility="gone" />

        <androidx.recyclerview.widget.RecyclerView
            android:id="@+id/recyclerRecords"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:nestedScrollingEnabled="false" />

    </LinearLayout>
</ScrollView>
```

### item_record.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.cardview.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_marginBottom="8dp"
    android:padding="12dp">

    <LinearLayout android:layout_width="match_parent" android:layout_height="wrap_content" android:orientation="vertical">
        <TextView android:id="@+id/tvDate" android:layout_width="match_parent" android:layout_height="wrap_content" android:textStyle="bold" />
        <TextView android:id="@+id/tvSummary" android:layout_width="match_parent" android:layout_height="wrap_content" />
        <TextView android:id="@+id/tvNotes" android:layout_width="match_parent" android:layout_height="wrap_content" />
    </LinearLayout>
</androidx.cardview.widget.CardView>
```

### activity_appointment.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="16dp">

    <LinearLayout android:layout_width="match_parent" android:layout_height="wrap_content" android:orientation="vertical">

        <EditText android:id="@+id/etDateTime" android:layout_width="match_parent" android:layout_height="wrap_content" android:hint="YYYY-MM-DD HH:mm" />
        <EditText android:id="@+id/etAddress" android:layout_width="match_parent" android:layout_height="wrap_content" android:hint="Endereço" />
        <EditText android:id="@+id/etNotes" android:layout_width="match_parent" android:layout_height="wrap_content" android:hint="Observações (opcional)" />

        <Button android:id="@+id/btnSave" android:layout_width="match_parent" android:layout_height="wrap_content" android:text="Salvar" />

    </LinearLayout>
</ScrollView>
```

### dialog_add_patient.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:padding="16dp">

    <EditText android:id="@+id/etName" android:layout_width="match_parent" android:layout_height="wrap_content" android:hint="Nome" />
    <EditText android:id="@+id/etBirth" android:layout_width="match_parent" android:layout_height="wrap_content" android:hint="Data de nascimento (opcional)" />
    <EditText android:id="@+id/etPhone" android:layout_width="match_parent" android:layout_height="wrap_content" android:hint="Telefone (opcional)" />
    <EditText android:id="@+id/etAddress" android:layout_width="match_parent" android:layout_height="wrap_content" android:hint="Endereço (opcional)" />

</LinearLayout>
```

### dialog_add_record.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:padding="16dp">

    <EditText android:id="@+id/etDate" android:layout_width="match_parent" android:layout_height="wrap_content" android:hint="Data e hora" />
    <EditText android:id="@+id/etSummary" android:layout_width="match_parent" android:layout_height="wrap_content" android:hint="Resumo (sintoma/diagnóstico)" />
    <EditText android:id="@+id/etNotes" android:layout_width="match_parent" android:layout_height="wrap_content" android:hint="Anotações do médico (opcional)" />

</LinearLayout>
```

---

## 6) Recursos (res/values)

### strings.xml
```xml
<resources>
    <string name="app_name">Doutor em Casa</string>
</resources>
```

### colors.xml
```xml
<resources>
    <color name="purple_200">#BB86FC</color>
    <color name="purple_500">#6200EE</color>
    <color name="purple_700">#3700B3</color>
    <color name="teal_200">#03DAC5</color>
    <color name="white">#FFFFFF</color>
</resources>
```

### themes/styles.xml
```xml
<resources>
    <style name="Theme.DoutorEmCasa" parent="Theme.MaterialComponents.DayNight.NoActionBar">
        <item name="colorPrimary">@color/purple_500</item>
        <item name="colorPrimaryVariant">@color/purple_700</item>
        <item name="colorOnPrimary">@color/white</item>
    </style>
</resources>
```

