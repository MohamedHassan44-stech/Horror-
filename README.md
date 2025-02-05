package com.horrorapp

import android.content.Intent
import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import android.widget.Button

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val btnStories = findViewById<Button>(R.id.btnStories)
        val btnHauntedMap = findViewById<Button>(R.id.btnHauntedMap)
        val btnARCamera = findViewById<Button>(R.id.btnARCamera)

        btnStories.setOnClickListener {
            startActivity(Intent(this, StoryListActivity::class.java))
        }
        btnHauntedMap.setOnClickListener {
            startActivity(Intent(this, HauntedMapActivity::class.java))
        }
        btnARCamera.setOnClickListener {
            startActivity(Intent(this, ARCameraActivity::class.java))
        }
    }
}

package com.horrorapp

import android.content.Intent
import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import android.widget.RecyclerView
import android.widget.Toast
import com.google.firebase.firestore.FirebaseFirestore

class StoryListActivity : AppCompatActivity() {
    private lateinit var recyclerView: RecyclerView
    private val db = FirebaseFirestore.getInstance()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_story_list)

        RecyclerView = findViewById(R.id.RecyclerViewStories)
        loadStories()
    }

    private fun loadStories() {
        db.collection("stories").get()
            .addOnSuccessListener { documents ->
                val stories = mutableListOf<String>()
                for (document in documents) {
                    stories.add(document.getString("title") ?: "بدون عنوان")
                }
                val adapter = StoryAdapter(this, storyList)
                RecyclerView.adapter = adapter

                RecyclerView.setOnItemClickListener { _, _, position, _ ->
                    val intent = Intent(this, StoryActivity::class.java)
                    intent.putExtra("storyTitle", stories[position])
                    startActivity(intent)
                }
            }
            .addOnFailureListener {
                Toast.makeText(this, "فشل تحميل القصص", Toast.LENGTH_SHORT).show()
            }
    }
}

package com.horrorapp

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import android.widget.TextView
import android.widget.Button
import android.widget.Toast
import com.google.firebase.firestore.FirebaseFirestore

class StoryActivity : AppCompatActivity() {
    private val db = FirebaseFirestore.getInstance()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_story)

        val storyText = findViewById<TextView>(R.id.storyText)
        val btnChoice1 = findViewById<Button>(R.id.btnChoice1)
        val btnChoice2 = findViewById<Button>(R.id.btnChoice2)

        val storyTitle = intent.getStringExtra("storyTitle")

        // تحميل القصة من Firebase
        db.collection("stories").whereEqualTo("title", storyTitle).get()
            .addOnSuccessListener { documents ->
                for (document in documents) {
                    storyText.text = document.getString("content") ?: "لا توجد تفاصيل"
                }
            }
            .addOnFailureListener {
                Toast.makeText(this, "حدث خطأ أثناء تحميل القصة", Toast.LENGTH_SHORT).show()
            }

        // التحكم في الاختيارات التفاعلية
        btnChoice1.setOnClickListener {
            storyText.text = "لقد اخترت الخيار الأول!"
        }
        btnChoice2.setOnClickListener {
            storyText.text = "لقد اخترت الخيار الثاني!"
        }
    }
}
package com.horrorapp

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import com.google.android.gms.maps.CameraUpdateFactory
import com.google.android.gms.maps.GoogleMap
import com.google.android.gms.maps.OnMapReadyCallback
import com.google.android.gms.maps.SupportMapFragment
import com.google.android.gms.maps.model.LatLng
import com.google.android.gms.maps.model.MarkerOptions

class HauntedMapActivity : AppCompatActivity(), OnMapReadyCallback {
    private lateinit var mMap: GoogleMap

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_map)

        val mapFragment = supportFragmentManager
            .findFragmentById(R.id.map) as SupportMapFragment
        mapFragment.getMapAsync(this)
    }

    override fun onMapReady(googleMap: GoogleMap) {
        mMap = googleMap
        val hauntedLocation = LatLng(30.0444, 31.2357) // موقع في القاهرة
        mMap.addMarker(MarkerOptions().position(hauntedLocation).title("مكان مسكون"))
        mMap.moveCamera(CameraUpdateFactory.newLatLngZoom(hauntedLocation, 10f))
    }
}
package com.horrorapp

import android.net.Uri
import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import android.widget.Toast
import com.google.ar.core.ArCoreApk
import com.google.ar.sceneform.AnchorNode
import com.google.ar.sceneform.rendering.ModelRenderable
import com.google.ar.sceneform.ux.ArFragment
import com.google.ar.sceneform.ux.TransformableNode

class ARCameraActivity : AppCompatActivity() {
    private lateinit var arFragment: ArFragment

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_ar_camera)

        // التحقق مما إذا كان الجهاز يدعم ARCore
        val availability = ArCoreApk.getInstance().checkAvailability(this)
        if (availability.isSupported) {
            arFragment = supportFragmentManager.findFragmentById(R.id.arFragment) as ArFragment

            // استماع لنقر المستخدم على سطح مستوٍ
            arFragment.setOnTapArPlaneListener { hitResult, _, _ ->
                val anchor = hitResult.createAnchor()

                // تحميل نموذج 3D بشكل غير متزامن مع إضافة فحص لوجود الملف في assets وتأثير وميض
val modelUri = Uri.parse("file:///android_asset/ghost_model.sfb")

if (!isAssetFileExists("ghost_model.sfb")) {
    Toast.makeText(this, "النموذج غير موجود في مجلد assets!", Toast.LENGTH_SHORT).show()
    return
}

ModelRenderable.builder()
    .setSource(this, modelUri)
    .build()
    .thenAccept { modelRenderable ->
        val anchorNode = AnchorNode(anchor)
        anchorNode.setParent(arFragment.arSceneView.scene)

        val transformableNode = TransformableNode(arFragment.transformationSystem)
        transformableNode.setParent(anchorNode)
        transformableNode.renderable = modelRenderable

        // تشغيل تأثير وميض عند تحميل النموذج
        if (transformableNode.renderable != null) {
            addFadeInAnimation(transformableNode)
        }
    }
    .exceptionally { throwable ->
        Toast.makeText(this, "فشل تحميل النموذج!", Toast.LENGTH_SHORT).show()
        Log.e("ARCameraActivity", "خطأ في تحميل النموذج: ${throwable.message}")
        null
    }

// دالة للتحقق مما إذا كان الملف موجودًا في مجلد assets
private fun isAssetFileExists(fileName: String): Boolean {
    return try {
        assets.open(fileName).close()
        true
    } catch (e: Exception) {
        false
    }
}

// دالة لإضافة تأثير وميض (Fade-in) عند تحميل النموذج
private fun addFadeInAnimation(transformableNode: TransformableNode) {
    val animator = ObjectAnimator.ofFloat(transformableNode, "alpha", 0f, 1f)
    animator.duration = 1500
    animator.interpolator = AccelerateDecelerateInterpolator()
    animator.start()
}


dependencies {
    implementation 'com.google.firebase:firebase-firestore:23.0.3'
    implementation 'com.google.firebase:firebase-auth:21.0.1'
    implementation 'com.google.ar:core:1.33.0'
}

class StoryAdapter(context: Context, private val storyList: List<Story>) : ArrayAdapter<Story>(context, 0, storyList) {
    override fun getView(position: Int, convertView: View?, parent: ViewGroup): View {
        val view = convertView ?: LayoutInflater.from(context).inflate(R.layout.item_story, parent, false)
        val story = storyList[position]

        val titleTextView = view.findViewById<TextView>(R.id.storyTitle)
        val descriptionTextView = view.findViewById<TextView>(R.id.storyDescription)

        titleTextView.text = story.title
        descriptionTextView.text = story.description

        return view
    }
}

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <RecyclerView
        android:id="@+id/RecyclerView"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />
</LinearLayout>

<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical"
    android:padding="16dp">

    <TextView
        android:id="@+id/storyTitle"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textSize="18sp"
        android:textStyle="bold" />

    <TextView
        android:id="@+id/storyDescription"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textSize="14sp"
        android:paddingTop="4dp" />
</LinearLayout>
val RecyclerView = findViewById<RecyclerView>(R.id.RecyclerView)
val storyList = listOf(
    Story("عنوان القصة 1", "وصف القصة 1"),
    Story("عنوان القصة 2", "وصف القصة 2")
)

val adapter = StoryAdapter(this, storyList)

<androidx.recyclerview.widget.RecyclerView
    android:id="@+id/recyclerView"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:scrollbars="vertical" />

recyclerView.adapter = adapter
recyclerView.layoutManager = LinearLayoutManager(this)

data class Story(val title: String, val description: String)
import android.content.Context
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.TextView
import android.widget.ArrayAdapter

private fun isNetworkAvailable(): Boolean {
    val connectivityManager = getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
    val activeNetwork = connectivityManager.activeNetwork
    return activeNetwork != null
}


private fun loadStories() {
   if (!isNetworkAvailable()) {
    Toast.makeText(this, "لا يوجد اتصال بالإنترنت! سيتم تحميل القصص المخزنة.", Toast.LENGTH_SHORT).show()
    loadOfflineStories()
    return
}


    db.collection("stories").get()
        .addOnSuccessListener { documents ->
            val stories = mutableListOf<Story>()
            for (document in documents) {
                val title = document.getString("title") ?: "بدون عنوان"
                val description = document.getString("description") ?: "لا يوجد وصف"
                stories.add(Story(title, description))
            }
            val adapter = StoryAdapter(this, storyList)
            recyclerView.adapter = adapter
        }
        .addOnFailureListener {
            Toast.makeText(this, "فشل تحميل القصص", Toast.LENGTH_SHORT).show()
        }
}

private fun loadStories() {
    progressBar.visibility = View.VISIBLE // إظهار التحميل
    if (!isNetworkAvailable()) {
    Toast.makeText(this, "لا يوجد اتصال بالإنترنت! سيتم تحميل القصص المخزنة.", Toast.LENGTH_SHORT).show()
    loadOfflineStories()
    return
}

private fun addFadeInAnimation(transformableNode: TransformableNode) {
    if (transformableNode.renderable == null) return

    val animator = ObjectAnimator.ofFloat(transformableNode, "alpha", 0f, 1f)
    animator.duration = 1500
    animator.interpolator = AccelerateDecelerateInterpolator()
    animator.start()
}


    db.collection("stories").get()
        .addOnSuccessListener { documents ->
            val stories = mutableListOf<Story>()
            for (document in documents) {
                val title = document.getString("title") ?: "بدون عنوان"
                val description = document.getString("description") ?: "لا يوجد وصف"
                stories.add(Story(title, description))
            }
            val adapter = StoryAdapter(this, storyList)
            recyclerView.adapter = adapter
            progressBar.visibility = View.GONE // إخفاء التحميل عند النجاح
        }
        .addOnFailureListener {
            Toast.makeText(this, "فشل تحميل القصص", Toast.LENGTH_SHORT).show()
            progressBar.visibility = View.GONE
        }
}

private var mediaPlayer: MediaPlayer? = null

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)

    // تشغيل المؤثر الصوتي عند بدء النشاط
    mediaPlayer = MediaPlayer.create(this, R.raw.scary_sound)
    mediaPlayer?.start()
}



import android.animation.ObjectAnimator
import android.media.MediaPlayer
import android.os.Bundle
import android.view.animation.LinearInterpolator
import androidx.appcompat.app.AppCompatActivity
import android.widget.Toast
import com.google.ar.sceneform.AnchorNode
import com.google.ar.sceneform.rendering.ModelRenderable
import com.google.ar.sceneform.ux.ArFragment
import com.google.ar.sceneform.ux.TransformableNode

class ARCameraActivity : AppCompatActivity() {
    private lateinit var arFragment: ArFragment
    private var mediaPlayer: MediaPlayer? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_ar_camera)

        arFragment = supportFragmentManager.findFragmentById(R.id.arFragment) as ArFragment

        // استماع لنقر المستخدم على سطح مستوٍ
        arFragment.setOnTapArPlaneListener { hitResult, _, _ ->
            val anchor = hitResult.createAnchor()

            // تشغيل الصوت عند ظهور النموذج
            if (mediaPlayer == null) {
                mediaPlayer = MediaPlayer.create(this, R.raw.scary_sound)
            }
            mediaPlayer?.let {
                if (!it.isPlaying) {
                    it.start()
                }
            }

            // تحميل نموذج 3D بشكل غير متزامن وإضافته إلى المشهد
            val modelUri = Uri.parse("file:///android_asset/ghost_model.sfb")

if (!isAssetFileExists("ghost_model.sfb")) {
    Toast.makeText(this, "النموذج غير موجود في مجلد assets!", Toast.LENGTH_SHORT).show()
    return
}

ModelRenderable.builder()
    .setSource(this, modelUri)
    .build()
    .thenAccept { modelRenderable ->
        val anchorNode = AnchorNode(anchor)
        anchorNode.setParent(arFragment.arSceneView.scene)

        val transformableNode = TransformableNode(arFragment.transformationSystem)
        transformableNode.setParent(anchorNode)
        transformableNode.renderable = modelRenderable

        // تشغيل تأثير وميض عند تحميل النموذج
        if (transformableNode.renderable != null) {
            addFadeInAnimation(transformableNode)
        }
    }
    .exceptionally { throwable ->
        Toast.makeText(this, "فشل تحميل النموذج!", Toast.LENGTH_SHORT).show()
        Log.e("ARCameraActivity", "خطأ في تحميل النموذج: ${throwable.message}")
        null
    }

// دالة للتحقق مما إذا كان الملف موجودًا في مجلد assets
private fun isAssetFileExists(fileName: String): Boolean {
    return try {
        assets.open(fileName).close()
        true
    } catch (e: Exception) {
        false
    }
}

// دالة لإضافة تأثير وميض عند تحميل النموذج
private fun addFadeInAnimation(transformableNode: TransformableNode) {
    val animator = ObjectAnimator.ofFloat(transformableNode, "alpha", 0f, 1f)
    animator.duration = 1500
    animator.interpolator = AccelerateDecelerateInterpolator()
    animator.start()
}

                
    override fun onDestroy() {
    super.onDestroy()

    // إلغاء أي تأثيرات `ObjectAnimator` قيد التشغيل
    if (::animator.isInitialized) {
        animator.cancel()
    }

    // التحقق مما إذا كان `MediaPlayer` لا يزال يعمل قبل تحريره
    mediaPlayer?.apply {
        if (it.isPlaying) {
            it.pause()
            }
        }
    }
    mediaPlayer?.release()
    mediaPlayer = null
}

// تشغيل الصوت عند ظهور النموذج
if (mediaPlayer == null) {
    mediaPlayer = MediaPlayer.create(this, R.raw.scary_sound)
}
mediaPlayer?.let {
    if (!it.isPlaying) {
        it.start()
    }
}

override fun onDestroy() {
    super.onDestroy()

    // التحقق مما إذا كان `MediaPlayer` لا يزال يعمل قبل تحريره
    mediaPlayer?.apply {
        if (isPlaying) {
            pause()
        }
    }
    mediaPlayer?.release()
    mediaPlayer = null
}


         // تحميل نموذج 3D بشكل غير متزامن وإضافته إلى المشهد
val modelUri = Uri.parse("file:///android_asset/ghost_model.sfb")

if (!isAssetFileExists("ghost_model.sfb")) {
    Toast.makeText(this, "النموذج غير موجود في مجلد assets!", Toast.LENGTH_SHORT).show()
    return
}

ModelRenderable.builder()
    .setSource(this, modelUri)
    .build()
    .thenAccept { modelRenderable ->
        val anchorNode = AnchorNode(anchor)
        anchorNode.setParent(arFragment.arSceneView.scene)

        val transformableNode = TransformableNode(arFragment.transformationSystem)
        transformableNode.setParent(anchorNode)
        transformableNode.renderable = modelRenderable

        // تشغيل تأثير وميض باستخدام ObjectAnimator بعد التأكد من تحميل النموذج
        if (transformableNode.renderable != null) {
            addFadeInAnimation(transformableNode)
        }
    }

private fun isAssetFileExists(fileName: String): Boolean {
    return try {
        assets.open(fileName).close()
        true
    } catch (e: Exception) {
        false
    }
}

    .exceptionally { throwable ->
        Toast.makeText(this, "فشل تحميل النموذج!", Toast.LENGTH_SHORT).show()
        Log.e("ARCameraActivity", "خطأ في تحميل النموذج: ${throwable.message}")
        null
    }

    // دالة لإضافة تأثير وميض (Fade-in) للنموذج ثلاثي الأبعاد
private fun addFadeInAnimation(transformableNode: TransformableNode) {
    val animator = ObjectAnimator.ofFloat(transformableNode, "alpha", 0f, 1f)
    animator.duration = 1500
    animator.interpolator = AccelerateDecelerateInterpolator()
    animator.start()
}



private var mediaPlayer: MediaPlayer? = null

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)

    val btnPlay = findViewById<Button>(R.id.btnPlay)
    val btnStop = findViewById<Button>(R.id.btnStop)

    btnPlay.setOnClickListener {
        if (mediaPlayer == null) {
            mediaPlayer = MediaPlayer.create(this, R.raw.scary_sound)
        }
        mediaPlayer?.let {
            if (!it.isPlaying) {
                it.start()
            }
        }
    }

    btnStop.setOnClickListener {
        mediaPlayer?.apply {
            if (it.isPlaying) {
                it.pause()
            }
        }
    }
}




import android.animation.ObjectAnimator
import android.view.animation.AccelerateDecelerateInterpolator
import com.google.ar.sceneform.ux.TransformableNode

// دالة لإضافة تأثير وميض (Fade-in) للنموذج ثلاثي الأبعاد
private fun addFadeInAnimation(transformableNode: TransformableNode) {
    // التأكد من أن النموذج جاهز قبل تطبيق التأثير
    if (transformableNode.renderable == null) return

    val animator = ObjectAnimator.ofFloat(transformableNode, "alpha", 0f, 1f)
    animator.duration = 1500
    animator.interpolator = AccelerateDecelerateInterpolator()
    animator.start()
}

// استدعاء الدالة عند تحميل النموذج
if (transformableNode.renderable != null) {
    addFadeInAnimation(transformableNode)
}
private fun isNetworkAvailable(context: Context): Boolean {
    val connectivityManager = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
    return connectivityManager.activeNetworkInfo?.isConnected == true
}
val soundList = listOf(R.raw.scary_sound1, R.raw.scary_sound2, R.raw.scary_sound3)
val randomSound = soundList.random()
mediaPlayer = MediaPlayer.create(this, randomSound)
mediaPlayer?.start()
private fun addFadeInAnimation(node: TransformableNode) {
    val animator = ObjectAnimator.ofFloat(node, "scaleX", 0f, 1f)
    animator.duration = 1500
    animator.start()
}

<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <ListView
        android:id="@+id/listViewStories"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_centerInParent="true"/>

    <ProgressBar
        android:id="@+id/progressBar"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerInParent="true"
        android:visibility="gone"/>
</RelativeLayout>
package com.horrorapp

import android.content.Intent
import android.os.Bundle
import android.view.View
import android.widget.ListView
import android.widget.ProgressBar
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import com.google.firebase.firestore.FirebaseFirestore

class StoryListActivity : AppCompatActivity() {
    private lateinit var listView: ListView
    private lateinit var progressBar: ProgressBar
    private val db = FirebaseFirestore.getInstance()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_story_list)

        listView = findViewById(R.id.listViewStories)
        progressBar = findViewById(R.id.progressBar)

        loadStories()
    }

    private fun loadStories() {
        progressBar.visibility = View.VISIBLE // إظهار الـ ProgressBar قبل تحميل البيانات

        db.collection("stories").get()
            .addOnSuccessListener { documents ->
                val stories = mutableListOf<String>()
                for (document in documents) {
                    stories.add(document.getString("title") ?: "بدون عنوان")
                }
                
                val adapter = StoryAdapter(this, stories)
                listView.adapter = adapter

                listView.setOnItemClickListener { _, _, position, _ ->
                    val intent = Intent(this, StoryActivity::class.java)
                    intent.putExtra("storyTitle", stories[position])
                    startActivity(intent)
                }

                progressBar.visibility = View.GONE // إخفاء الـ ProgressBar بعد التحميل
            }
            .addOnFailureListener {
                progressBar.visibility = View.GONE // إخفاء الـ ProgressBar في حالة الفشل
                Toast.makeText(this, "فشل تحميل القصص", Toast.LENGTH_SHORT).show()
            }
    }
}
package com.horrorapp

data class Story(
    val title: String = "",
    val description: String = ""
)
package com.horrorapp

import androidx.lifecycle.LiveData
import androidx.lifecycle.MutableLiveData
import com.google.firebase.firestore.FirebaseFirestore

class StoryRepository {
    private val db = FirebaseFirestore.getInstance()

    private val _stories = MutableLiveData<List<Story>>()  
    val stories: LiveData<List<Story>> get() = _stories  

    fun fetchStories() {
        db.collection("stories").get()
            .addOnSuccessListener { documents ->
                val storyList = mutableListOf<Story>()
                for (document in documents) {
                    val title = document.getString("title") ?: "بدون عنوان"
                    val description = document.getString("description") ?: "لا يوجد وصف"
                    storyList.add(Story(title, description))
                }
                _stories.value = storyList  
            }
            .addOnFailureListener {
                _stories.value = emptyList()  
            }
    }
}
package com.horrorapp

import androidx.lifecycle.LiveData
import androidx.lifecycle.ViewModel

class StoryViewModel : ViewModel() {
    private val repository = StoryRepository()
    val stories: LiveData<List<Story>> = repository.stories  

    init {
        repository.fetchStories()  
    }
}
package com.horrorapp

import android.content.Intent
import android.os.Bundle
import android.view.View
import android.widget.ListView
import android.widget.ProgressBar
import android.widget.Toast
import androidx.activity.viewModels
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.Observer

class StoryListActivity : AppCompatActivity() {
    private lateinit var listView: ListView
    private lateinit var progressBar: ProgressBar
    private val viewModel: StoryViewModel by viewModels()  

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_story_list)

        listView = findViewById(R.id.listViewStories)
        progressBar = findViewById(R.id.progressBar)

        progressBar.visibility = View.VISIBLE  

        viewModel.stories.observe(this, Observer { stories ->
            progressBar.visibility = View.GONE  
            if (stories.isEmpty()) {
                Toast.makeText(this, "فشل تحميل القصص", Toast.LENGTH_SHORT).show()
            } else {
                val adapter = StoryAdapter(this, stories)
                listView.adapter = adapter

                listView.setOnItemClickListener { _, _, position, _ ->
                    val intent = Intent(this, StoryActivity::class.java)
                    intent.putExtra("storyTitle", stories[position].title)
                    startActivity(intent)
                }
            }
        })
    }
}
