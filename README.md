rootProject.name = "AppMarket"
include(":app")
plugins {
    kotlin("multiplatform") apply false
}
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
}

android {
    namespace 'com.example.appmarket'
    compileSdk 34

    defaultConfig {
        applicationId "com.example.appmarket"
        minSdk 21
        targetSdk 34
        versionCode 1
        versionName "1.0"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    compileOptions { sourceCompatibility JavaVersion.VERSION_17; targetCompatibility JavaVersion.VERSION_17 }
    kotlinOptions { jvmTarget = "17" }
    buildFeatures { compose true }
    composeOptions { kotlinCompilerExtensionVersion '1.5.3' } // örnek sürüm, Android Studio güncel sürümüne göre ayarla
}

dependencies {
    implementation "androidx.core:core-ktx:1.12.0"
    implementation "androidx.lifecycle:lifecycle-runtime-ktx:2.6.2"
    implementation "androidx.activity:activity-compose:1.8.0"
    implementation "androidx.compose.ui:ui:1.5.3"
    implementation "androidx.compose.material:material:1.5.3"
    implementation "androidx.compose.ui:ui-tooling-preview:1.5.3"

    // Navigation
    implementation "androidx.navigation:navigation-compose:2.7.0"

    // Retrofit + Moshi (API client)
    implementation "com.squareup.retrofit2:retrofit:2.9.0"
    implementation "com.squareup.retrofit2:converter-moshi:2.9.0"
    implementation "com.squareup.moshi:moshi-kotlin:1.15.0"

    // Coroutines
    implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3"

    // Coil for image loading (Compose)
    implementation "io.coil-kt:coil-compose:2.4.0"

    // ViewModel (Compose)
    implementation "androidx.lifecycle:lifecycle-viewmodel-compose:2.6.2"

    debugImplementation "androidx.compose.ui:ui-tooling:1.5.3"
}
<manifest package="com.example.appmarket" xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET"/>
    <application
        android:allowBackup="true"
        android:label="AppMarket"
        android:icon="@mipmap/ic_launcher"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.AppMarket">
        <activity android:name=".MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
package com.example.appmarket

import android.content.Intent
import android.net.Uri
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.material.MaterialTheme
import androidx.compose.material.Surface
import androidx.lifecycle.viewmodel.compose.viewModel
import com.example.appmarket.ui.AppNavHost
import com.example.appmarket.ui.theme.AppMarketTheme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            AppMarketTheme {
                Surface(color = MaterialTheme.colors.background) {
                    AppNavHost(onOpenInPlayStore = { packageName ->
                        // Play Store veya tarayıcı ile aç
                        try {
                            val intent = Intent(Intent.ACTION_VIEW, Uri.parse("market://details?id=$packageName"))
                            startActivity(intent)
                        } catch (e: Exception) {
                            val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://play.google.com/store/apps/details?id=$packageName"))
                            startActivity(intent)
                        }
                    })
                }
            }
        }
    }
}
package com.example.appmarket.model

data class AppInfo(
    val id: String,
    val name: String,
    val shortDescription: String,
    val iconUrl: String,
    val packageName: String,
    val rating: Float
)
package com.example.appmarket.network

import com.example.appmarket.model.AppInfo
import retrofit2.http.GET

interface ApiService {
    @GET("apps")
    suspend fun getApps(): List<AppInfo>
}
package com.example.appmarket.repository

import com.example.appmarket.model.AppInfo
import com.example.appmarket.network.ApiService
import kotlinx.coroutines.delay

class AppRepository(private val api: ApiService?) {
    // Eğer api null ise fake data döndür
    suspend fun fetchApps(): List<AppInfo> {
        api?.let {
            return it.getApps()
        }
        // Fake data
        delay(400) // simüle
        return listOf(
            AppInfo("1","Sample App 1","Kısa açıklama 1","https://via.placeholder.com/150","com.example.sample1",4.5f),
            AppInfo("2","Sample App 2","Kısa açıklama 2","https://via.placeholder.com/150","com.example.sample2",4.0f),
            AppInfo("3","Sample App 3","Kısa açıklama 3","https://via.placeholder.com/150","com.example.sample3",3.8f)
        )
    }
}
package com.example.appmarket.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.appmarket.model.AppInfo
import com.example.appmarket.repository.AppRepository
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch

class AppListViewModel(private val repo: AppRepository): ViewModel() {
    private val _apps = MutableStateFlow<List<AppInfo>>(emptyList())
    val apps: StateFlow<List<AppInfo>> = _apps

    fun loadApps() {
        viewModelScope.launch {
            try {
                val list = repo.fetchApps()
                _apps.value = list
            } catch (e: Exception) {
                _apps.value = emptyList()
            }
        }
    }
}
package com.example.appmarket.ui

import androidx.compose.runtime.Composable
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.navigation.NavType
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import androidx.navigation.navArgument
import com.example.appmarket.ui.screens.DetailScreen
import com.example.appmarket.ui.screens.ListScreen
import com.example.appmarket.viewmodel.AppListViewModel
import com.example.appmarket.repository.AppRepository
import com.example.appmarket.network.ApiService
import retrofit2.Retrofit
import retrofit2.converter.moshi.MoshiConverterFactory

@Composable
fun AppNavHost(onOpenInPlayStore: (String)->Unit) {
    val navController = rememberNavController()

    // Basit DI: Retrofit veya null
    val retrofit = Retrofit.Builder()
        .baseUrl("https://your-api.example.com/") // kendi API adresin
        .addConverterFactory(MoshiConverterFactory.create())
        .build()
    val api = try { retrofit.create(ApiService::class.java) } catch (e: Exception) { null }
    val repo = AppRepository(api)
    val vm: AppListViewModel = viewModel(factory = AppListViewModelFactory(repo))

    NavHost(navController = navController, startDestination = "list") {
        composable("list") {
            ListScreen(vm) { selected ->
                navController.navigate("detail/${selected.id}")
            }
        }
        composable(
            "detail/{appId}",
            arguments = listOf(navArgument("appId"){ type = NavType.StringType })
        ) { backStackEntry ->
            val id = backStackEntry.arguments?.getString("appId") ?: ""
            val app = vm.apps.value.find { it.id == id }
            if (app != null) {
                DetailScreen(app, onOpenInPlayStore)
            } else {
                // fallback
            }
        }
    }
}
package com.example.appmarket.ui.screens

import androidx.compose.foundation.layout.*
import androidx.compose.material.Button
import androidx.compose.material.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import coil.compose.AsyncImage
import com.example.appmarket.model.AppInfo

@Composable
fun DetailScreen(app: AppInfo, onOpenInPlayStore: (String)->Unit) {
    Column(modifier = Modifier.fillMaxSize().padding(16.dp)) {
        AsyncImage(model = app.iconUrl, contentDescription = app.name, modifier = Modifier.size(120.dp))
        Spacer(Modifier.height(8.dp))
        Text(app.name)
        Spacer(Modifier.height(4.dp))
        Text(app.shortDescription)
        Spacer(Modifier.height(8.dp))
        Text("Rating: ${app.rating}")
        Spacer(Modifier.height(20.dp))
        Button(onClick = { onOpenInPlayStore(app.packageName) }) {
            Text("Aç / Play Store'da Gör")
        }
    }
}
