╔╗╔══╗╔╗╔╗╔══╗╔╗╔══╗  
║║║╔═╝║║║║║╔╗║║║║╔═╝  
║╚╝║──  ║╚╝║║║║║║╚╝║──  
║╔╗║──  ╚═╗║║║║║║╔╗║──  
║║║╚═╗ ─╔╝║║║║║║║║╚═╗  
╚╝╚══╝─ ╚═╝╚╝╚╝╚╝╚══╝  
Авторство: Михаил Спиркин
# Документация по разработке Android приложения с использованием Kotlin и Bottom Navigation, и файлов

Файлы храняться (у меня почему-то, хотя раньше всё норм было)) ) в  /data/путь приложения/files/Documents

## Архитектура файлов
com.example.studcon  
├── Adapter  
│   ├── SpecialtyAdapter.kt  
│   ├── SpecialtyViewAdapter.kt  
│   ├── UserAdapter.kt  
│   └── SpecialtyViewAdapter.kt  
├── Data  
│   ├── Specialty.kt  
│   └── User.kt  
├── ViewModel  
│   └── SharedViewModel.kt  
├── ui  
│   ├── login  
│   │   └── LoginActivity.kt  
│   ├── SpecialtiesFragment  
│   │   └── SpecialtiesFragment.kt  
│   ├── SubmitScoreFragment  
│   │   └── SubmitScoreFragment.kt  
│   ├── ResultsFragment  
│   │   └── ResultsFragment.kt  
│   └── MainActivity.kt  
└── res  
    ├── layout  
    │   ├── activity_main.xml  
    │   ├── fragment_specialties.xml  
    │   ├── fragment_submitscore.xml  
    │   ├── fragment_results.xml  
    │   ├── item_specialty.xml  
    │   ├── item_user.xml  
    │   └── item_specialty_view.xml  
    └── menu  
        └── bottom_nav_menu.xml  
## Шаги по разработке приложения

### 1. Создание нового проекта в Android Studio
- Откройте Android Studio и выберите "New Project".
- Выберите "Bottom Navigation Activity" и задайте имя проекта (например, `StudCon`).
- Убедитесь, что язык выбран как Kotlin.

### 2. Добавление зависимостей
В `build.gradle` (Module: app) добавьте необходимые зависимости:
```groovy
dependencies {
    implementation 'androidx.navigation:navigation-fragment-ktx:2.5.0'
    implementation 'androidx.navigation:navigation-ui-ktx:2.5.0'
    implementation 'androidx.appcompat:appcompat:1.4.0'
    implementation 'com.google.android.material:material:1.5.0'
    implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.4.1'
    implementation 'androidx.recyclerview:recyclerview:1.2.1'
    implementation 'pub.devrel:easypermissions:3.0.0'
}
```
### 3. Создание Авторизации
`LoginActivity.kt`
``` 
package com.example.studcon.ui.login

import android.Manifest
import android.content.Context
import android.content.Intent
import android.os.Bundle
import android.os.Environment
import androidx.annotation.StringRes
import androidx.appcompat.app.AppCompatActivity
import android.widget.Toast
import com.example.studcon.MainActivity
import com.example.studcon.databinding.ActivityLoginBinding
import com.example.studcon.R
import pub.devrel.easypermissions.EasyPermissions
import java.io.File
import java.io.FileOutputStream

class LoginActivity : AppCompatActivity(), EasyPermissions.PermissionCallbacks {
    private lateinit var binding: ActivityLoginBinding

    private fun createFileInDocuments(context: Context, fileName: String, content: String) {
        val file = File(context.getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS), fileName)
        file.writeText(content) // Записываем содержимое в файл
    }

    private val REQUEST_CODE_PERMISSIONS = 1001

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        binding = ActivityLoginBinding.inflate(layoutInflater)
        setContentView(binding.root)

        // Запрашиваем разрешения на доступ к хранилищу
        requestPermissions()

        // Устанавливаем обработчик нажатия кнопки "Войти"
        binding.login.setOnClickListener {
            val user = binding.username.text.toString()
            val pass = binding.password.text.toString()
            if (authenticate(user, pass)) {
                // Успешная аутентификация
                Toast.makeText(this, "Успешный вход", Toast.LENGTH_SHORT).show()
                var isAdmin:Boolean = false
                if (user == "admin" || pass == "admin"){
                    isAdmin = true
                }

                val intent = Intent(this, MainActivity::class.java).apply {
                    putExtra("IS_ADMIN", isAdmin) // Передаем информацию о том, что пользователь администратор
                }

                startActivity(intent)
                finish() // Закрываем LoginActivity, чтобы не возвращаться назад
            } else {
                // Ошибка аутентификации, добавляем нового пользователя
                addUser (user, pass)
                showLoginFailed(R.string.login_failed)
            }
        }
    }

    private fun showLoginFailed(@StringRes errorString: Int) {
        Toast.makeText(applicationContext, errorString, Toast.LENGTH_SHORT).show()
    }

    private fun requestPermissions() {
        if (EasyPermissions.hasPermissions(this, Manifest.permission.WRITE_EXTERNAL_STORAGE, Manifest.permission.READ_EXTERNAL_STORAGE)) {

        } else {
            EasyPermissions.requestPermissions(
                this,
                "Необходимы разрешения на доступ к хранилищу",
                REQUEST_CODE_PERMISSIONS,
                Manifest.permission.WRITE_EXTERNAL_STORAGE,
                Manifest.permission.READ_EXTERNAL_STORAGE
            )
        }
        initializeFiles()
    }

    private fun authenticate(username: String, password: String): Boolean {


        val userFile = getUserFile()

        if (!userFile.exists()) {
            return false
        }

        var isAuthenticated = false // Флаг для отслеживания успешной аутентификации

        userFile.forEachLine { line ->
            val trimmedLine = line.trim()
            if (trimmedLine.isNotEmpty()) {
                val (fileUsername, filePassword) = trimmedLine.split(",")
                if (fileUsername.trim() == username && filePassword.trim() == password) {
                    isAuthenticated = true // Успешная аутентификация
                }
            }
        }

        return isAuthenticated // Возвращаем результат
    }

    private fun addUser (username: String, password: String) {
        val userFileName = "users.txt"
        val userFileContent = "$username,$password\n"

        // Путь к файлу в папке "Документы"
        val file = File(getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS), userFileName)

        try {
            // Используем FileOutputStream с параметром append = true для добавления данных
            FileOutputStream(file, true).use { outputStream ->
                outputStream.write(userFileContent.toByteArray())
            }
            Toast.makeText(this, "Пользователь добавлен", Toast.LENGTH_SHORT).show()
        } catch (e: Exception) {
            Toast.makeText(this, "Ошибка при добавлении пользователя", Toast.LENGTH_SHORT).show()
        }

        // Успешная аутентификация
        Toast.makeText(this, "Успешный вход", Toast.LENGTH_SHORT).show()

        val intent = Intent(this, MainActivity::class.java)
        startActivity(intent)
        finish() // Закрываем LoginActivity, чтобы не возвращаться назад
    }

    private fun getUserFile(): File {
        // Изменяем путь к файлу на папку "Документы"
        return File(getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS), "users.txt")
    }

    private fun initializeFiles() {
        // Создаем файл пользователей, если он не существует
        val userFileName = "users.txt"
        val userFileContent = "admin,admin\n"

        val userFile = File(getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS), userFileName)
        if (!userFile.exists()) {
            createFileInDocuments(this, userFileName, userFileContent)
        }

        // Список специальностей
        val specialties = listOf(
            "09.02.07 - Программирование в компьютерных системах",
            "13.01.10 - Информационные системы и программирование",
            "13.02.10 - Информационные технологии",
            "15.02.10 - Автоматизация и управление",
            "15.02.12 - Техническое обслуживание и ремонт радиоэлектронной аппаратуры",
            "15.02.15 - Компьютерные системы и комплексы",
            "22.02.02 - Технология машиностроения",
            "22.02.05 - Технология металлообработки",
            "38.02.01 - Экономика"
        )

        for (specialty in specialties) {
            val specialtyFileName = "$specialty.txt"
            val specialtyFile = File(getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS), specialtyFileName)

            // Создаем файл специальности, если он не существует
            if (!specialtyFile.exists()) {
                createFileInDocuments(this, specialtyFileName, "0.5")
            }
        }
    }

    override fun onPermissionsGranted(requestCode: Int, perms: List<String>) {
        if (requestCode == REQUEST_CODE_PERMISSIONS) {
            initializeFiles()
        }
    }

    override fun onPermissionsDenied(requestCode: Int, perms: List<String>) {
        if (requestCode == REQUEST_CODE_PERMISSIONS) {
            Toast.makeText(this, "Разрешение на запись в хранилище не предоставлено", Toast.LENGTH_SHORT).show()
        }
    }
}
```

`login_activity.xml`
```
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/container"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingBottom="@dimen/activity_vertical_margin"
    tools:context=".ui.login.LoginActivity">

    <EditText
        android:id="@+id/username"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginTop="96dp"
        android:hint="@string/prompt_email"
        android:inputType="textEmailAddress"
        android:selectAllOnFocus="true"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <EditText
        android:id="@+id/password"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_marginTop="8dp"
        android:hint="@string/prompt_password"
        android:imeActionLabel="@string/action_sign_in_short"
        android:imeOptions="actionDone"
        android:inputType="textPassword"
        android:selectAllOnFocus="true"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/username" />

    <Button
        android:id="@+id/login"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="start"
        android:layout_marginTop="16dp"
        android:layout_marginBottom="64dp"
        android:enabled="true"
        android:text="@string/action_sign_in"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/password"
        app:layout_constraintVertical_bias="0.2" />

    <ProgressBar
        android:id="@+id/loading"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:layout_marginTop="64dp"
        android:layout_marginBottom="64dp"
        android:visibility="gone"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="@+id/password"
        app:layout_constraintStart_toStartOf="@+id/password"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintVertical_bias="0.3" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

`AndroidManifest.xml`

```
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
...
<activity
            android:name=".ui.login.LoginActivity"
            android:exported="true"
            android:label="@string/title_activity_login" >
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <activity
            android:name=".MainActivity"
            android:exported="true" android:theme="@style/Theme.StudCon.NoActionBar"
            android:label="@string/app_name">

        </activity>
```

### 4. Создание основной активности и макета
`MainActivity.kt`
```
package com.example.studcon

import android.os.Bundle
import android.os.Environment
import android.view.View
import android.widget.ArrayAdapter
import android.widget.Toast
import androidx.appcompat.app.ActionBarDrawerToggle
import com.google.android.material.bottomnavigation.BottomNavigationView
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.ViewModelProvider
import androidx.navigation.findNavController
import androidx.navigation.ui.AppBarConfiguration
import androidx.navigation.ui.navigateUp
import androidx.navigation.ui.setupActionBarWithNavController
import androidx.navigation.ui.setupWithNavController
import com.example.studcon.Data.Specialty
import com.example.studcon.ViewModel.SharedViewModel
import com.example.studcon.databinding.ActivityMainBinding
import java.io.File

class MainActivity : AppCompatActivity() {

    private lateinit var binding: ActivityMainBinding

    private lateinit var sharedViewModel: SharedViewModel

    private lateinit var specialties: List<Specialty>

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        // Настройка Toolbar
        setSupportActionBar(binding.toolbar)

        // Получаем информацию о том, является ли пользователь администратором
        val isAdmin = intent.getBooleanExtra("IS_ADMIN", false)
        val navView: BottomNavigationView = binding.navView

        val navController = findNavController(R.id.nav_host_fragment_activity_main)

        // Passing each menu ID as a set of Ids because each
        // menu should be considered as top level destinations.
        val appBarConfiguration = AppBarConfiguration(
            setOf(
                R.id.navigation_Specialties, R.id.navigation_SubmitScore, R.id.navigation_Results
            )
        )

        setupActionBarWithNavController(navController, appBarConfiguration)
        navView.setupWithNavController(navController)

        // Проверяем, является ли пользователь администратором
        if (isAdmin) {
            binding.ln.visibility = View.VISIBLE // Делаем LinearLayout видимым
        } else {
            binding.ln.visibility = View.GONE // Скрываем LinearLayout, если не администратор
        }

        sharedViewModel = ViewModelProvider(this).get(SharedViewModel::class.java)

        // Наблюдаем за изменениями в списке специальностей
        sharedViewModel.specialties.observe(this) { specialtiesList ->
            specialties = specialtiesList
            setupSpinner()
        }

        binding.buttonUpdatePassingScore.setOnClickListener {
            val selectedSpecialty = binding.spinnerSpecialty.selectedItem.toString()
            val passingScoreInput = binding.editTextPassingScore.text.toString()

            if (passingScoreInput.isNotEmpty()) {
                val passingScore = passingScoreInput.toDouble()
                updatePassingScore(selectedSpecialty, passingScore)
                Toast.makeText(this, "Проходной балл обновлен", Toast.LENGTH_SHORT).show()
            } else {
                Toast.makeText(this, "Пожалуйста, введите проходной балл", Toast.LENGTH_SHORT).show()
            }
        }

        // Обработка нажатия на кнопку для открытия боковой панели
        binding.navViewDrawer.setNavigationItemSelectedListener { menuItem ->
            // Здесь можно добавить обработку нажатий, если это необходимо
            true
        }

        // Добавьте обработку нажатия на кнопку "гамбургер"
        val toggle = ActionBarDrawerToggle(
            this, binding.drawerLayout, binding.toolbar,
            R.string.navigation_drawer_open, R.string.navigation_drawer_close
        )
        binding.drawerLayout.addDrawerListener(toggle)
        toggle.syncState()
    }
    private fun setupSpinner() {
        val specialtyNames = specialties.map { it.name }
        val adapter = ArrayAdapter(this, android.R.layout.simple_spinner_item, specialtyNames)
        adapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item)
        binding.spinnerSpecialty.adapter = adapter
    }

    private fun updatePassingScore(specialty: String, passingScore: Double) {
        val fileName = "$specialty.txt"
        val file = File(this.getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS), fileName)

        try {
            // Сначала создаем новый файл с новым проходным баллом
            val newFile = File(file.parent, "temp_$fileName")
            newFile.writeText("$passingScore\n") // Записываем проходной балл

            // Читаем старые данные и добавляем их после проходного балла
            if (file.exists()) {
                file.forEachLine { line ->
                    newFile.appendText("$line\n")
                }
            }

            // Удаляем старый файл и переименовываем новый
            file.delete()
            newFile.renameTo(file)
        } catch (e: Exception) {
            Toast.makeText(this,"Ошибка при обновлении проходного балла",Toast.LENGTH_SHORT).show()
        }
    }
    override fun onSupportNavigateUp(): Boolean {
        val navController = findNavController(R.id.nav_host_fragment_activity_main)
        return navController.navigateUp(binding.drawerLayout) || super.onSupportNavigateUp()
    }
}
```

` activity_main.xml `

``` <androidx.drawerlayout.widget.DrawerLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/drawer_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.constraintlayout.widget.ConstraintLayout
        android:id="@+id/container"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <androidx.appcompat.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            android:background="@color/black" />

        <com.google.android.material.bottomnavigation.BottomNavigationView
            android:id="@+id/nav_view"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            app:layout_constraintBottom_toBottomOf="parent"
            app:menu="@menu/bottom_nav_menu" />

        <fragment
            android:id="@+id/nav_host_fragment"
            android:name="androidx.navigation.fragment.NavHostFragment"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            app:defaultNavHost="true"
            app:navGraph="@navigation/mobile_navigation" />
    </androidx.constraintlayout.widget.ConstraintLayout>

    <com.google.android.material.navigation.NavigationView
        android:id="@+id/nav_view_drawer"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:layout_gravity="start">
        <!-- Здесь добавьте элементы навигации -->
    </com.google.android.material.navigation.NavigationView>

</androidx.drawerlayout.widget.DrawerLayout> 
```

## 5. Создание фрагментов

Для дальнейшего функционала необходимо переименновать автосгенерированные фрагменты в названия по нашему
(можете оставить и так но будет удобнее в понимании и чтении кода)

`SpecialtiesFragment.kt`
```
package com.example.studcon.ui.SpecialtiesFragment

import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import androidx.fragment.app.Fragment
import androidx.lifecycle.ViewModelProvider
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import com.example.studcon.Adapter.SpecialtyAdapter
import com.example.studcon.Data.Specialty
import com.example.studcon.ViewModel.SharedViewModel
import com.example.studcon.databinding.FragmentSpecialtiesBinding

class SpecialtiesFragment : Fragment() {

    private var _binding: FragmentSpecialtiesBinding? = null
    private lateinit var recyclerView: RecyclerView
    private lateinit var specialtyAdapter: SpecialtyAdapter
    private lateinit var sharedViewModel: SharedViewModel
    // This property is only valid between onCreateView and
    // onDestroyView.
    private val binding get() = _binding!!

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {

        _binding = FragmentSpecialtiesBinding.inflate(inflater, container, false)
        val root: View = binding.root

        sharedViewModel = ViewModelProvider(requireActivity()).get(SharedViewModel::class.java)

        recyclerView = binding.recyclerViewSpecialties
        recyclerView.layoutManager = LinearLayoutManager(context)

        // Пример списка специальностей
        val specialties = listOf(
            Specialty("09.02.07 - Программирование в компьютерных системах",
                "Специальность, связанная с разработкой программного обеспечения, включая создание, тестирование и поддержку программных приложений."),
            Specialty("13.01.10 - Информационные системы и программирование",
                "Специальность, охватывающая проектирование, разработку и внедрение информационных систем и программного обеспечения."),
            Specialty("13.02.10 - Информационные технологии",
                "Специальность, изучающая методы и средства обработки, хранения и передачи информации с использованием современных технологий."),
            Specialty("15.02.10 - Автоматизация и управление",
                "Специальность, связанная с проектированием и эксплуатацией автоматизированных систем управления и контроля."),
            Specialty("15.02.12 - Техническое обслуживание и ремонт радиоэлектронной аппаратуры",
                "Специальность, охватывающая диагностику, обслуживание и ремонт радиоэлектронных устройств и систем."),
            Specialty("15.02.15 - Компьютерные системы и комплексы",
                "Специальность, изучающая проектирование, эксплуатацию и обслуживание компьютерных систем и сетей."),
            Specialty("22.02.02 - Технология машиностроения",
                "Специальность, связанная с проектированием, производством и эксплуатацией машин и механизмов."),
            Specialty("22.02.05 - Технология металлообработки",
                "Специальность, охватывающая процессы обработки металлов, включая механическую и термическую обработку."),
            Specialty("38.02.01 - Экономика",
                "Специальность, изучающая экономические процессы, управление и анализ в различных сферах экономики.")
        )

// Обновляем ViewModel
        sharedViewModel.specialties.value = specialties

        specialtyAdapter = SpecialtyAdapter(specialties)
        recyclerView.adapter = specialtyAdapter
        return root
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

`SubmitScoreFragment.kt`
```
package com.example.studcon.ui.SubmitScoreFragment

import android.R
import android.os.Bundle
import android.os.Environment
import android.util.Log
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.ArrayAdapter
import android.widget.TextView
import android.widget.Toast
import androidx.fragment.app.Fragment
import androidx.lifecycle.ViewModelProvider
import com.example.studcon.Data.Specialty
import com.example.studcon.ViewModel.SharedViewModel
import com.example.studcon.databinding.FragmentSubmitscoreBinding
import java.io.File
import java.io.FileOutputStream

class SubmitScoreFragment : Fragment() {

    private var _binding: FragmentSubmitscoreBinding? = null

    private lateinit var sharedViewModel: SharedViewModel

    private lateinit var specialties: List<Specialty>

    // This property is only valid between onCreateView and
    // onDestroyView.
    private val binding get() = _binding!!

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {

        _binding = FragmentSubmitscoreBinding.inflate(inflater, container, false)
        val root: View = binding.root

        sharedViewModel = ViewModelProvider(requireActivity()).get(SharedViewModel::class.java)

        // Наблюдаем за изменениями в списке специальностей
        sharedViewModel.specialties.observe(viewLifecycleOwner) { specialtiesList ->
            specialties = specialtiesList
            setupSpinner()
        }


        binding.buttonSubmit.setOnClickListener {
            saveData()
        }

        return root
    }

    private fun setupSpinner() {
        val specialtyNames = specialties.map { it.name }
        val adapter = ArrayAdapter(requireContext(), R.layout.simple_spinner_item, specialtyNames)
        adapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item)
        binding.spinnerSpecialty.adapter = adapter
    }

    private fun saveData() {
        val fullName = binding.editTextFullName.text.toString()
        val selectedSpecialty = binding.spinnerSpecialty.selectedItem.toString()
        val averageScore = binding.editTextAverageScore.text.toString()

        if (fullName.isEmpty() || averageScore.isEmpty()) {
            Toast.makeText(requireContext(), "Пожалуйста, заполните все поля", Toast.LENGTH_SHORT).show()
            return
        }

        // Формируем строку для сохранения
        val dataToSave = "$fullName, $averageScore\n"

        // Имя файла специальности
        val specialtyFileName = "$selectedSpecialty.txt"

        // Путь к файлу в папке "Документы"
        val file = File(requireContext().getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS), specialtyFileName)

        try {
            FileOutputStream(file, true).use { outputStream ->
                outputStream.write(dataToSave.toByteArray())
            }
            Toast.makeText(requireContext(), "Данные успешно сохранены", Toast.LENGTH_SHORT).show()
            clearFields()
        } catch (e: Exception) {
            Toast.makeText(requireContext(), "Ошибка при сохранении данных", Toast.LENGTH_SHORT).show()
        }
    }

    private fun clearFields() {
        binding.editTextFullName.text.clear()
        binding.editTextAverageScore.text.clear()
        binding.spinnerSpecialty.setSelection(0)
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

`ResultsFragment.kt`
```
package com.example.studcon.ui.ResultsFragment

import android.os.Bundle
import android.os.Environment
import android.util.Log
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.TextView
import android.widget.Toast
import androidx.fragment.app.Fragment
import androidx.lifecycle.ViewModelProvider
import androidx.recyclerview.widget.LinearLayoutManager
import com.example.studcon.Adapter.SpecialtyAdapter
import com.example.studcon.Adapter.SpecialtyViewAdapter
import com.example.studcon.Adapter.UserAdapter
import com.example.studcon.Data.Specialty
import com.example.studcon.Data.User
import com.example.studcon.ViewModel.SharedViewModel
import com.example.studcon.databinding.FragmentResultsBinding
import java.io.File
import java.util.UUID

class ResultsFragment : Fragment() {

    private var _binding: FragmentResultsBinding? = null

    private lateinit var specialties: List<Specialty> // Список всех специальностей
    private lateinit var users: List<User> // Список пользователей для выбранной специальности
    private var passingScore: Double = 0.0 // Проходной балл

    private lateinit var sharedViewModel: SharedViewModel
    // This property is only valid between onCreateView and
    // onDestroyView.
    private val binding get() = _binding!!

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {

        _binding = FragmentResultsBinding.inflate(inflater, container, false)
        val root: View = binding.root

        sharedViewModel = ViewModelProvider(requireActivity()).get(SharedViewModel::class.java)

        // Наблюдаем за изменениями в списке специальностей
        sharedViewModel.specialties.observe(viewLifecycleOwner) { specialtiesList ->
            specialties = specialtiesList
            setupSpecialtyAdapter()
        }

        // Инициализация списка пользователей
        users = listOf() // Здесь вы можете загрузить пользователей для выбранной специальности



        return root
    }

    private fun setupSpecialtyAdapter() {
        val specialtyAdapter = SpecialtyViewAdapter(specialties) { selectedSpecialty ->
            // Обработка клика по специальности
            loadUsersForSpecialty(selectedSpecialty)
        }
        binding.recyclerViewSpecialties.adapter = specialtyAdapter
        binding.recyclerViewSpecialties.layoutManager = LinearLayoutManager(requireContext(), LinearLayoutManager.HORIZONTAL, false)
    }

    private fun loadUsersForSpecialty(specialty: Specialty) {
        users = fetchUsersForSpecialty(specialty.name)
        // Настройка адаптера для пользователей
        val userAdapter = UserAdapter(users, passingScore)
        binding.recyclerViewUsers.adapter = userAdapter
        binding.recyclerViewUsers.layoutManager = LinearLayoutManager(requireContext())
        (binding.recyclerViewUsers.adapter as UserAdapter).updateUsers(users)
    }

    private fun fetchUsersForSpecialty(specialtyCode: String): List<User> {
        val usersList = mutableListOf<User>()
        val specialtyFileName = "$specialtyCode.txt"
        val file = File(requireContext().getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS), specialtyFileName)

        if (file.exists()) {
            // Читаем все строки из файла
            val lines = file.readLines()

            // Итерируем по строкам с индексами
            lines.forEachIndexed { index, line ->
                if (index == 0) {
                    passingScore = line.toDoubleOrNull() ?: 0.0 // Считываем проходной балл
                } else {
                    val parts = line.split(", ")
                    if (parts.size == 2) {
                        val fullName = parts[0]
                        val averageScore = parts[1].toDoubleOrNull() ?: 0.0
                        usersList.add(User(UUID.randomUUID().toString(), fullName, averageScore, specialtyCode, specialtyCode))
                    }
                }
            }
        } else {
            Log.d("ResultsFragment", "Файл $specialtyFileName не найден.")
        }
        return usersList
    }



    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

`specialties_fragment.xml`
```
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".ui.SpecialtiesFragment.SpecialtiesFragment">

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recycler_view_specialties"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:padding="16dp" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

`submitScore_fragment.xml`
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <EditText
        android:id="@+id/editTextFullName"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="ФИО" />

    <Spinner
        android:id="@+id/spinnerSpecialty"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

    <EditText
        android:id="@+id/editTextAverageScore"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Средний балл аттестата"
        android:inputType="numberDecimal" />

    <Button
        android:id="@+id/buttonSubmit"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Отправить" />

</LinearLayout>
```

`results_fragment.xml`
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:orientation="vertical"
    android:padding="16dp">

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerViewSpecialties"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginBottom="16dp"
        android:orientation="horizontal"
        app:layoutManager="androidx.recyclerview.widget.LinearLayoutManager"/>

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerViewUsers"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" />
</LinearLayout>
```

## 6. Создание ViewModel
`SharedViewModel`
```
package com.example.studcon.ViewModel

import androidx.lifecycle.MutableLiveData
import androidx.lifecycle.ViewModel
import com.example.studcon.Data.Specialty

class SharedViewModel : ViewModel() {
    val specialties = MutableLiveData<List<Specialty>>()
}
```

## 7. Создание DataClass

```
data class Specialty(val name: String, val description: String)
```
```
data class User(val id: String, val fullName: String, val averageScore: Double)
```

## 8. Создание Adapter's
Разметку самих элементов сами сделаете, ничего сложного)
(ПРИМЕР: holder.specialtyName.text = specialty.name -> есть TextView c id specialtyName)

```
package com.example.studcon.Adapter

import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.TextView
import androidx.recyclerview.widget.RecyclerView
import com.example.studcon.Data.Specialty
import com.example.studcon.R

class SpecialtyAdapter(private val specialties: List<Specialty>) : RecyclerView.Adapter<SpecialtyAdapter.SpecialtyViewHolder>() {

    class SpecialtyViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val specialtyName: TextView = itemView.findViewById(R.id.specialty_name)
        val specialtyDescription: TextView = itemView.findViewById(R.id.specialty_description)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): SpecialtyViewHolder {
        val view = LayoutInflater.from(parent.context).inflate(R.layout.item_specialty, parent, false)
        return SpecialtyViewHolder(view)
    }

    override fun onBindViewHolder(holder: SpecialtyViewHolder, position: Int) {
        val specialty = specialties[position]
        holder.specialtyName.text = specialty.name
        holder.specialtyDescription.text = specialty.description
    }

    override fun getItemCount(): Int {
        return specialties.size
    }
}
```

```
package com.example.studcon.Adapter

import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.TextView
import androidx.recyclerview.widget.RecyclerView
import com.example.studcon.Data.Specialty
import com.example.studcon.R

class SpecialtyAdapter(private val specialties: List<Specialty>) : RecyclerView.Adapter<SpecialtyAdapter.SpecialtyViewHolder>() {

    class SpecialtyViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val specialtyName: TextView = itemView.findViewById(R.id.specialty_name)
        val specialtyDescription: TextView = itemView.findViewById(R.id.specialty_description)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): SpecialtyViewHolder {
        val view = LayoutInflater.from(parent.context).inflate(R.layout.item_specialty, parent, false)
        return SpecialtyViewHolder(view)
    }

    override fun onBindViewHolder(holder: SpecialtyViewHolder, position: Int) {
        val specialty = specialties[position]
        holder.specialtyName.text = specialty.name
        holder.specialtyDescription.text = specialty.description
    }

    override fun getItemCount(): Int {
        return specialties.size
    }
}
```

```
package com.example.studcon.Adapter

import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.TextView
import androidx.recyclerview.widget.RecyclerView
import com.example.studcon.Data.Specialty
import com.example.studcon.R

class SpecialtyAdapter(private val specialties: List<Specialty>) : RecyclerView.Adapter<SpecialtyAdapter.SpecialtyViewHolder>() {

    class SpecialtyViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val specialtyName: TextView = itemView.findViewById(R.id.specialty_name)
        val specialtyDescription: TextView = itemView.findViewById(R.id.specialty_description)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): SpecialtyViewHolder {
        val view = LayoutInflater.from(parent.context).inflate(R.layout.item_specialty, parent, false)
        return SpecialtyViewHolder(view)
    }

    override fun onBindViewHolder(holder: SpecialtyViewHolder, position: Int) {
        val specialty = specialties[position]
        holder.specialtyName.text = specialty.name
        holder.specialtyDescription.text = specialty.description
    }

    override fun getItemCount(): Int {
        return specialties.size
    }
}
```
