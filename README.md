
## AndroidManifest.xml

```
<manifest package="com.example.photofilter"
    xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />

    <application
        android:allowBackup="true"
        android:label="PhotoFilter"
        android:theme="@style/Theme.AppCompat.Light.NoActionBar">
        <activity android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

---

## res/layout/activity_main.xml

```
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <!-- CameraX Preview -->
    <androidx.camera.view.PreviewView
        android:id="@+id/previewView"
        android:layout_width="0dp"
        android:layout_height="0dp"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toTopOf="@+id/filterList"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

    <!-- GPUImageView overlay used to show filters on top of preview -->
    <jp.co.cyberagent.android.gpuimage.GPUImageView
        android:id="@+id/gpuImageView"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:keepScreenOn="true"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toTopOf="@+id/filterList"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

    <!-- Filter list -->
    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/filterList"
        android:layout_width="0dp"
        android:layout_height="120dp"
        android:padding="8dp"
        app:layout_constraintBottom_toTopOf="@+id/bottomBar"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />

    <!-- Bottom bar controls -->
    <LinearLayout
        android:id="@+id/bottomBar"
        android:layout_width="0dp"
        android:layout_height="80dp"
        android:orientation="horizontal"
        android:gravity="center"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent">

        <Button
            android:id="@+id/btnCapture"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Capture" />

        <Button
            android:id="@+id/btnRecord"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Record" />

    </LinearLayout>

</androidx.constraintlayout.widget.ConstraintLayout>
```

---

## res/layout/item_filter.xml

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="wrap_content"
    android:layout_height="match_parent"
    android:padding="6dp">

    <ImageView
        android:id="@+id/thumb"
        android:layout_width="80dp"
        android:layout_height="80dp"
        android:scaleType="centerCrop" />

    <TextView
        android:id="@+id/title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textSize="12sp"
        android:ellipsize="end" />
</LinearLayout>
```

---

## MainActivity.kt

```
package com.example.photofilter

import android.Manifest
import android.content.ContentValues
import android.content.pm.PackageManager
import android.graphics.Bitmap
import android.net.Uri
import android.os.Build
import android.os.Bundle
import android.provider.MediaStore
import android.util.Log
import android.view.View
import android.widget.Button
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.camera.core.ImageCapture
nimport androidx.camera.core.ImageCaptureException
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.camera.view.PreviewView
import androidx.camera.core.Preview
import androidx.camera.video.*
import androidx.core.content.ContextCompat
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import jp.co.cyberagent.android.gpuimage.GPUImageView
import jp.co.cyberagent.android.gpuimage.filter.*
import java.io.OutputStream
import java.text.SimpleDateFormat
import java.util.*
import java.util.concurrent.ExecutorService
import java.util.concurrent.Executors

class MainActivity : AppCompatActivity() {

    private lateinit var previewView: PreviewView
    private lateinit var gpuImageView: GPUImageView
    private lateinit var filterList: RecyclerView
    private lateinit var btnCapture: Button
    private lateinit var btnRecord: Button

    private var imageCapture: ImageCapture? = null

    // Video
    private var recording: Recording? = null
    private var videoCapture: VideoCapture<Recorder>? = null

    private lateinit var cameraExecutor: ExecutorService

    private val filters = listOf(
        Pair("Normal", null),
        Pair("Sepia", GPUImageSepiaFilter()),
        Pair("Contrast", GPUImageContrastFilter(1.2f)),
        Pair("Saturation", GPUImageSaturationFilter(1.3f)),
        Pair("Vintage", GPUImageLookupFilter()) // placeholder; you can create LUT-based filter
    )

    private val requestPermissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { permissions ->
        val granted = permissions.all { it.value }
        if (granted) startCamera() else finish()
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        previewView = findViewById(R.id.previewView)
        gpuImageView = findViewById(R.id.gpuImageView)
        filterList = findViewById(R.id.filterList)
        btnCapture = findViewById(R.id.btnCapture)
        btnRecord = findViewById(R.id.btnRecord)

        cameraExecutor = Executors.newSingleThreadExecutor()

        filterList.layoutManager = LinearLayoutManager(this, LinearLayoutManager.HORIZONTAL, false)
        filterList.adapter = FilterAdapter(filters.map { it.first }) { idx ->
            val filter = filters[idx].second
            runOnUiThread {
                gpuImageView.filter = filter
            }
        }

        btnCapture.setOnClickListener { takePhoto() }
        btnRecord.setOnClickListener { toggleRecording() }

        checkPermissionsAndStart()
    }

    private fun checkPermissionsAndStart() {
        val required = mutableListOf(Manifest.permission.CAMERA, Manifest.permission.RECORD_AUDIO)
        if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.P) {
            required.add(Manifest.permission.WRITE_EXTERNAL_STORAGE)
        }
        val notGranted = required.filter { ContextCompat.checkSelfPermission(this, it) != PackageManager.PERMISSION_GRANTED }
        if (notGranted.isEmpty()) {
            startCamera()
        } else {
            requestPermissionLauncher.launch(notGranted.toTypedArray())
        }
    }

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)
        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()

            val preview = Preview.Builder().build().also {
                it.setSurfaceProvider(previewView.surfaceProvider)
            }

            imageCapture = ImageCapture.Builder().build()

            // Video setup
            val recorder = Recorder.Builder().setQualitySelector(QualitySelector.from(Quality.HIGHEST)).build()
            videoCapture = VideoCapture.withOutput(recorder)

            try {
                cameraProvider.unbindAll()
                cameraProvider.bindToLifecycle(this, androidx.camera.core.CameraSelector.DEFAULT_BACK_CAMERA, preview, imageCapture, videoCapture)
            } catch (e: Exception) {
                Log.e("MainActivity", "Use case binding failed", e)
            }

        }, ContextCompat.getMainExecutor(this))
    }

    private fun takePhoto() {
        val imageCapture = imageCapture ?: return

        // We'll capture a still frame and then apply GPUImage filter on the captured bitmap.
        imageCapture.takePicture(ContextCompat.getMainExecutor(this), object : ImageCapture.OnImageCapturedCallback() {
            override fun onCaptureSuccess(imageProxy: androidx.camera.core.ImageProxy) {
                try {
                    val bitmap = ImageUtil.imageProxyToBitmap(imageProxy)
                    runOnUiThread {
                        // Apply current GPU filter to bitmap
                        gpuImageView.setImage(bitmap)
                        val filtered = gpuImageView.bitmapWithFilterApplied
                        saveBitmapToGallery(filtered)
                    }
                } finally {
                    imageProxy.close()
                }
            }

            override fun onError(exception: ImageCaptureException) {
                Log.e("MainActivity", "Photo capture failed: ${exception.message}")
            }
        })
    }

    private fun saveBitmapToGallery(bitmap: Bitmap) {
        val filename = "IMG_${SimpleDateFormat("yyyyMMdd_HHmmss", Locale.US).format(Date())}.jpg"
        val fos: OutputStream?
        val contentValues = ContentValues().apply {
            put(MediaStore.MediaColumns.DISPLAY_NAME, filename)
            put(MediaStore.MediaColumns.MIME_TYPE, "image/jpeg")
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                put(MediaStore.MediaColumns.RELATIVE_PATH, "DCIM/PhotoFilterApp")
            }
        }
        val resolver = contentResolver
        val uri: Uri? = resolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, contentValues)
        fos = uri?.let { resolver.openOutputStream(it) }
        fos?.use { out ->
            bitmap.compress(Bitmap.CompressFormat.JPEG, 95, out)
        }
    }

    private fun toggleRecording() {
        val videoCapture = this.videoCapture ?: return
        if (recording != null) {
            recording?.stop()
            recording = null
            btnRecord.text = "Record"
            return
        }

        val name = "VID_${SimpleDateFormat("yyyyMMdd_HHmmss", Locale.US).format(Date())}.mp4"
        val contentValues = ContentValues().apply {
            put(MediaStore.MediaColumns.DISPLAY_NAME, name)
            put(MediaStore.MediaColumns.MIME_TYPE, "video/mp4")
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                put(MediaStore.MediaColumns.RELATIVE_PATH, "DCIM/PhotoFilterApp")
            }
        }
        val mediaStoreOutput = MediaStoreOutputOptions.Builder(contentResolver, MediaStore.Video.Media.EXTERNAL_CONTENT_URI)
            .setContentValues(contentValues)
            .build()

        recording = videoCapture.output
            .prepareRecording(this, mediaStoreOutput)
            .apply { if (ContextCompat.checkSelfPermission(this@MainActivity, Manifest.permission.RECORD_AUDIO) == PackageManager.PERMISSION_GRANTED) withAudioEnabled() }
            .start(ContextCompat.getMainExecutor(this)) { recordEvent ->
                when (recordEvent) {
                    is VideoRecordEvent.Finalize -> {
                        if (!recordEvent.hasError()) {
                            Log.d("MainActivity", "Video saved: ${recordEvent.outputResults.outputUri}")
                        } else {
                            Log.e("MainActivity", "Video error: ${recordEvent.error}")
                        }
                    }
                    else -> {}
                }
            }
        btnRecord.text = "Stop"
    }

    override fun onDestroy() {
        super.onDestroy()
        cameraExecutor.shutdown()
    }
}
```

---

## FilterAdapter.kt

```
package com.example.photofilter

import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.ImageView
import android.widget.TextView
import androidx.recyclerview.widget.RecyclerView

class FilterAdapter(private val items: List<String>, private val onClick: (Int) -> Unit) : RecyclerView.Adapter<FilterAdapter.VH>() {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
        val v = LayoutInflater.from(parent.context).inflate(R.layout.item_filter, parent, false)
        return VH(v)
    }

    override fun onBindViewHolder(holder: VH, position: Int) {
        holder.title.text = items[position]
        holder.itemView.setOnClickListener { onClick(position) }
    }

    override fun getItemCount(): Int = items.size

    class VH(v: View) : RecyclerView.ViewHolder(v) {
        val thumb: ImageView = v.findViewById(R.id.thumb)
        val title: TextView = v.findViewById(R.id.title)
    }
}
```

---

## FileUtils / ImageUtil helper

```
package com.example.photofilter

import android.graphics.Bitmap
import android.graphics.ImageFormat
import android.graphics.Rect
import android.graphics.YuvImage
import androidx.camera.core.ImageProxy
import java.io.ByteArrayOutputStream

object ImageUtil {
    fun imageProxyToBitmap(image: ImageProxy): Bitmap {
        val plane = image.planes[0]
        val yBuffer = image.planes[0].buffer
        val uBuffer = image.planes[1].buffer
        val vBuffer = image.planes[2].buffer

        val ySize = yBuffer.remaining()
        val uSize = uBuffer.remaining()
        val vSize = vBuffer.remaining()

        val nv21 = ByteArray(ySize + uSize + vSize)
        yBuffer.get(nv21, 0, ySize)
        vBuffer.get(nv21, ySize, vSize)
        uBuffer.get(nv21, ySize + vSize, uSize)

        val yuvImage = YuvImage(nv21, ImageFormat.NV21, image.width, image.height, null)
        val out = ByteArrayOutputStream()
        yuvImage.compressToJpeg(Rect(0, 0, image.width, image.height), 90, out)
        val imageBytes = out.toByteArray()
        return android.graphics.BitmapFactory.decodeByteArray(imageBytes, 0, imageBytes.size)
    }
}
```

---

## Notes & next steps

1. **Realtime filtered video**: Applying GPUImage filters to recorded video in realtime requires rendering the camera frames through an OpenGL pipeline into a Surface that the MediaRecorder reads, or encoding the GPU-rendered frames manually. This is advanced; if you want it, I can add a full pipeline using `SurfaceTexture` + `EGL` and `MediaCodec`/`MediaMuxer` or use a library that supports GPU filters for video (ffmpeg with shader processing / custom GL pipeline). Say "video filters please" and I'll add that.

2. **iPhone-style filters**: The sample filters included are basic (sepia, contrast, saturation). For more iPhone-like LUTs, include PNG LUT assets and use `GPUImageLookupFilter` with those LUT bitmaps.

3. **Performance**: Always test on target devices. Use a background thread for expensive operations and reduce preview resolution if needed.

4. **Permissions**: The code requests runtime permissions. On Android 13+ you may want to use the new photo/video permissions (`READ_MEDIA_IMAGES` / `READ_MEDIA_VIDEO`).

5. **To import**: Create a new Android Studio project, replace the project-level and app-level gradle files and copy the Kotlin files and layouts into `app/src/main/...`.

---

If you want, I can now:
- Add ready-to-import zip (I can create files here for you to download), or
- Implement LUT-based iPhone-style filters with LUT assets, or
- Add realtime filtered video recording (more code but possible).

Tell me which of the three you want next and I will extend the project.
