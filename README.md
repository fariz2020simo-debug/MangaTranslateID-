import os, textwrap, zipfile

base = "/mnt/data/MangaTranslateID"
os.makedirs(base + "/app/src/main/java/com/fariz/mangatranslateid", exist_ok=True)
os.makedirs(base + "/app/src/main/res/values", exist_ok=True)

files = {
"settings.gradle.kts": """
pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories { google(); mavenCentral() }
}
rootProject.name = "MangaTranslateID"
include(":app")
""",
"build.gradle.kts": """
plugins {
    id("com.android.application") version "8.6.1" apply false
    id("org.jetbrains.kotlin.android") version "2.0.21" apply false
}
""",
"gradle.properties": """
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official
""",
"app/build.gradle.kts": """
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.fariz.mangatranslateid"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.fariz.mangatranslateid"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "0.1.0"
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions { jvmTarget = "17" }
}

dependencies {
    implementation("androidx.core:core-ktx:1.15.0")
    implementation("androidx.appcompat:appcompat:1.7.0")
    implementation("com.google.mlkit:text-recognition:16.0.1")
}
""",
"app/src/main/AndroidManifest.xml": """
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET" />
    <application
        android:allowBackup="true"
        android:label="MangaTranslate ID"
        android:theme="@style/AppTheme"
        android:usesCleartextTraffic="false">
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:configChanges="orientation|screenSize|keyboardHidden">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
""",
"app/src/main/res/values/styles.xml": """
<resources>
    <style name="AppTheme" parent="Theme.AppCompat.DayNight.NoActionBar">
        <item name="android:fontFamily">sans</item>
        <item name="android:colorAccent">#8B5CF6</item>
        <item name="android:statusBarColor">#111827</item>
        <item name="android:navigationBarColor">#111827</item>
        <item name="android:windowLightStatusBar">false</item>
    </style>
</resources>
""",
"app/src/main/java/com/fariz/mangatranslateid/MainActivity.kt": r"""
package com.fariz.mangatranslateid

import android.annotation.SuppressLint
import android.graphics.Bitmap
import android.graphics.Canvas
import android.graphics.Color
import android.graphics.Paint
import android.graphics.RectF
import android.os.Bundle
import android.view.Gravity
import android.view.View
import android.view.ViewGroup
import android.webkit.WebChromeClient
import android.webkit.WebView
import android.webkit.WebViewClient
import android.widget.Button
import android.widget.FrameLayout
import android.widget.LinearLayout
import android.widget.ProgressBar
import android.widget.TextView
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import com.google.mlkit.vision.common.InputImage
import com.google.mlkit.vision.text.TextRecognition
import com.google.mlkit.vision.text.latin.TextRecognizerOptions
import org.json.JSONObject
import java.net.HttpURLConnection
import java.net.URLEncoder
import java.net.URL
import java.util.concurrent.Executors

data class TranslationBox(val rect: RectF, val text: String)

class MainActivity : AppCompatActivity() {
    private lateinit var webView: WebView
    private lateinit var overlay: TranslationOverlay
    private lateinit var status: TextView
    private lateinit var translateButton: Button
    private val executor = Executors.newSingleThreadExecutor()
    private val recognizer = TextRecognition.getClient(TextRecognizerOptions.DEFAULT_OPTIONS)
    private var busy = false

    @SuppressLint("SetJavaScriptEnabled")
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val root = FrameLayout(this).apply { setBackgroundColor(Color.BLACK) }
        webView = WebView(this).apply {
            settings.javaScriptEnabled = true
            settings.domStorageEnabled = true
            settings.loadsImagesAutomatically = true
            settings.useWideViewPort = true
            settings.loadWithOverviewMode = true
            webViewClient = WebViewClient()
            webChromeClient = WebChromeClient()
            loadUrl("https://mangadex.org/")
        }
        root.addView(webView, FrameLayout.LayoutParams(-1, -1))

        overlay = TranslationOverlay(this)
        overlay.visibility = View.GONE
        root.addView(overlay, FrameLayout.LayoutParams(-1, -1))

        val controls = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            gravity = Gravity.END
            setPadding(12, 12, 12, 12)
        }
        status = TextView(this).apply {
            text = "Buka bab komik, lalu tekan Terjemahkan"
            setTextColor(Color.WHITE)
            textSize = 12f
            setBackgroundColor(0xCC111827.toInt())
            setPadding(12, 8, 12, 8)
        }
        translateButton = Button(this).apply {
            text = "Terjemahkan halaman"
            setOnClickListener { translateVisiblePage() }
        }
        val clearButton = Button(this).apply {
            text = "Sembunyikan terjemahan"
            setOnClickListener {
                overlay.setBoxes(emptyList())
                overlay.visibility = View.GONE
                webView.visibility = View.VISIBLE
                status.text = "Terjemahan disembunyikan"
            }
        }
        controls.addView(status)
        controls.addView(translateButton)
        controls.addView(clearButton)
        val controlParams = FrameLayout.LayoutParams(
            ViewGroup.LayoutParams.WRAP_CONTENT,
            ViewGroup.LayoutParams.WRAP_CONTENT,
            Gravity.BOTTOM or Gravity.END
        ).apply { bottomMargin = 8; rightMargin = 8 }
        root.addView(controls, controlParams)
        setContentView(root)
    }

    private fun translateVisiblePage() {
        if (busy) return
        busy = true
        translateButton.isEnabled = false
        status.text = "Membaca teks pada halaman..."
        try {
            val bitmap = Bitmap.createBitmap(
                webView.width.coerceAtLeast(1),
                webView.height.coerceAtLeast(1),
                Bitmap.Config.ARGB_8888
            )
            webView.draw(Canvas(bitmap))
            val image = InputImage.fromBitmap(bitmap, 0)
            recognizer.process(image)
                .addOnSuccessListener { result ->
                    val lines = result.textBlocks.flatMap { it.lines }
                        .filter { it.text.trim().length >= 2 }
                        .take(35)
                    if (lines.isEmpty()) {
                        finishWithMessage("Tidak menemukan teks. Pastikan halaman komik sudah tampil.")
                        bitmap.recycle()
                        return@addOnSuccessListener
                    }
                    status.text = "Menerjemahkan ${lines.size} bagian..."
                    val boxes = mutableListOf<TranslationBox>()
                    var remaining = lines.size
                    lines.forEach { line ->
                        executor.execute {
                            val translated = translateText(line.text)
                            runOnUiThread {
                                if (translated.isNotBlank() && translated != line.text) {
                                    boxes.add(
                                        TranslationBox(
                                            RectF(line.boundingBox ?: android.graphics.Rect()),
                                            translated
                                        )
                                    )
                                }
                                remaining--
                                if (remaining == 0) {
                                    overlay.setBoxes(boxes.toList())
                                    overlay.visibility = View.VISIBLE
                                    status.text = "Selesai: ${boxes.size} bagian diterjemahkan"
                                    finishBusy()
                                    bitmap.recycle()
                                }
                            }
                        }
                    }
                }
                .addOnFailureListener {
                    bitmap.recycle()
                    finishWithMessage("OCR gagal. Coba muat ulang halaman dan ulangi.")
                }
        } catch (e: Exception) {
            finishWithMessage("Gagal membaca halaman: ${e.message ?: "kesalahan tidak diketahui"}")
        }
    }

    private fun translateText(text: String): String {
        var connection: HttpURLConnection? = null
        return try {
            // MyMemory public endpoint is convenient for a prototype; it has rate limits.
            val encoded = URLEncoder.encode(text, "UTF-8")
            val url = URL("https://api.mymemory.translated.net/get?q=$encoded&langpair=en|id")
            connection = (url.openConnection() as HttpURLConnection).apply {
                requestMethod = "GET"
                connectTimeout = 8000
                readTimeout = 10000
                setRequestProperty("User-Agent", "MangaTranslateID/0.1")
            }
            if (connection.responseCode !in 200..299) return ""
            val body = connection.inputStream.bufferedReader().use { it.readText() }
            JSONObject(body).optJSONObject("responseData")?.optString("translatedText", "") ?: ""
        } catch (_: Exception) {
            ""
        } finally {
            connection?.disconnect()
        }
    }

    private fun finishWithMessage(message: String) {
        runOnUiThread {
            status.text = message
            Toast.makeText(this, message, Toast.LENGTH_LONG).show()
            finishBusy()
        }
    }

    private fun finishBusy() {
        busy = false
        translateButton.isEnabled = true
    }

    override fun onDestroy() {
        recognizer.close()
        webView.destroy()
        executor.shutdownNow()
        super.onDestroy()
    }
}

class TranslationOverlay(context: android.content.Context) : View(context) {
    private val boxes = mutableListOf<TranslationBox>()
    private val fill = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = 0xF2FFF9C4.toInt()
        style = Paint.Style.FILL
    }
    private val border = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = 0xFF5B4B00.toInt()
        style = Paint.Style.STROKE
        strokeWidth = 1.5f
    }
    private val textPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = Color.BLACK
        textSize = 15f
        textAlign = Paint.Align.LEFT
    }

    fun setBoxes(newBoxes: List<TranslationBox>) {
        boxes.clear()
        boxes.addAll(newBoxes)
        invalidate()
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        boxes.forEach { item ->
            val r = RectF(item.rect)
            r.left = r.left.coerceAtLeast(0f)
            r.top = r.top.coerceAtLeast(0f)
            r.right = r.right.coerceAtMost(width.toFloat())
            r.bottom = r.bottom.coerceAtMost(height.toFloat())
            if (r.width() < 4f || r.height() < 4f) return@forEach

            // Draw a readable translation card near the original text position.
            val padding = 5f
            val maxWidth = (r.width() * 1.8f).coerceIn(90f, 280f)
            val wrapped = wrapText(item.text, textPaint, maxWidth - padding * 2)
            val lineHeight = textPaint.textSize * 1.15f
            val cardHeight = (wrapped.size * lineHeight + padding * 2)
                .coerceAtMost((height - r.top).coerceAtLeast(24f))
            val card = RectF(
                r.left,
                r.top,
                (r.left + maxWidth).coerceAtMost(width.toFloat()),
                r.top + cardHeight
            )
            canvas.drawRoundRect(card, 5f, 5f, fill)
            canvas.drawRoundRect(card, 5f, 5f, border)
            wrapped.forEachIndexed { index, line ->
                val y = card.top + padding + textPaint.textSize + index * lineHeight
                if (y < card.bottom) canvas.drawText(line, card.left + padding, y, textPaint)
            }
        }
    }

    private fun wrapText(text: String, paint: Paint, maxWidth: Float): List<String> {
        val words = text.replace('\n', ' ').split(Regex("\\s+")).filter { it.isNotBlank() }
        val result = mutableListOf<String>()
        var line = ""
        for (word in words) {
            val candidate = if (line.isEmpty()) word else "$line $word"
            if (paint.measureText(candidate) <= maxWidth || line.isEmpty()) {
                line = candidate
            } else {
                result.add(line)
                line = word
            }
        }
        if (line.isNotEmpty()) result.add(line)
        return result.ifEmpty { listOf(text.take(28)) }
    }
}
""",
"README.md": """
# MangaTranslate ID — Android prototype

Prototipe Android untuk membuka MangaDex dan menerjemahkan teks Inggris yang terlihat di layar menjadi bahasa Indonesia.

## Yang sudah ada
- WebView untuk membuka https://mangadex.org/
- Tombol untuk OCR teks di halaman yang terlihat
- ML Kit Text Recognition untuk mengenali tulisan
- MyMemory Translation API untuk terjemahan Inggris → Indonesia
- Lapisan kartu terjemahan di atas halaman

## Batasan penting
- Ini prototipe, bukan APK siap pakai yang sudah diuji.
- Tombol menerjemahkan halaman yang sedang terlihat, bukan menerjemahkan semua halaman secara terus-menerus.
- OCR bisa salah membaca font manga, teks vertikal, efek suara, atau tulisan kecil.
- Terjemahan tampil sebagai kartu di atas gambar; aplikasi belum menghapus tulisan Inggris asli.
- Endpoint terjemahan publik dapat membatasi jumlah permintaan, tidak menjamin privasi, dan dapat berubah. Jangan gunakan untuk konten sensitif.
- MangaDex dapat mengubah halaman, login, atau perlindungan web sehingga perilaku WebView bisa berbeda.

## Membangun APK
1. Instal Android Studio pada komputer (build APK langsung dari HP tidak dijamin).
2. Ekstrak ZIP ini.
3. Android Studio → Open → pilih folder `MangaTranslateID`.
4. Tunggu Gradle Sync. Jika diminta, instal Android SDK Platform 35 dan JDK 17.
5. Pilih Build → Build Bundle(s) / APK(s) → Build APK(s).
6. APK debug biasanya ada di `app/build/outputs/apk/debug/app-debug.apk`.
7. Pindahkan APK ke HP Android, buka file APK, lalu izinkan instalasi dari sumber tersebut bila diminta.

## Catatan build
Proyek menggunakan Android Gradle Plugin 8.6.1, Kotlin 2.0.21, dan AndroidX. Build pertama membutuhkan internet untuk mengunduh dependensi.

## Privasi
Halaman yang tampil diproses di perangkat untuk OCR. Potongan teks yang dikenali dikirim ke layanan terjemahan MyMemory melalui internet. Kode ini tidak meminta izin overlay sistem karena terjemahan digambar di dalam aplikasi sendiri.
""",
}
for rel, content in files.items():
    path = os.path.join(base, rel)
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, "w", encoding="utf-8") as f:
        f.write(textwrap.dedent(content).lstrip())

zip_path = "/mnt/data/MangaTranslateID-Android-Project.zip"
with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as z:
    for root, dirs, filenames in os.walk(base):
        for filename in filenames:
            full = os.path.join(root, filename)
            z.write(full, os.path.relpath(full, "/mnt/data"))

print("Project files created:")
print(zip_path)
print("Included:", ", ".join(files.keys()))
