name: Build Jarvis APK
on:
  push:
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: 8.7
      - name: Create Jarvis project
        run: |
          mkdir -p jv/app/src/main/java/com/jarvis/app
          cd jv
          cat > settings.gradle.kts <<'EOF'
          pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
          dependencyResolutionManagement { repositories { google(); mavenCentral() } }
          rootProject.name = "Jarvis"
          include(":app")
          EOF
          cat > build.gradle.kts <<'EOF'
          plugins {
            id("com.android.application") version "8.5.2" apply false
            id("org.jetbrains.kotlin.android") version "1.9.24" apply false
          }
          EOF
          echo "org.gradle.jvmargs=-Xmx2g" > gradle.properties
          cat > app/build.gradle.kts <<'EOF'
          plugins { id("com.android.application"); id("org.jetbrains.kotlin.android") }
          android {
            namespace = "com.jarvis.app"
            compileSdk = 34
            defaultConfig { applicationId = "com.jarvis.app"; minSdk = 26; targetSdk = 34; versionCode = 1; versionName = "1.0" }
            compileOptions { sourceCompatibility = JavaVersion.VERSION_17; targetCompatibility = JavaVersion.VERSION_17 }
            kotlinOptions { jvmTarget = "17" }
          }
          EOF
          cat > app/src/main/AndroidManifest.xml <<'EOF'
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
            <uses-permission android:name="android.permission.INTERNET"/>
            <uses-permission android:name="android.permission.RECORD_AUDIO"/>
            <uses-permission android:name="android.permission.READ_CONTACTS"/>
            <uses-permission android:name="android.permission.CALL_PHONE"/>
            <uses-permission android:name="com.android.alarm.permission.SET_ALARM"/>
            <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
            <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MICROPHONE"/>
            <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
            <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
            <queries>
              <intent><action android:name="android.intent.action.MAIN"/><category android:name="android.intent.category.LAUNCHER"/></intent>
              <intent><action android:name="android.speech.RecognitionService"/></intent>
            </queries>
            <application android:label="Jarvis" android:theme="@android:style/Theme.DeviceDefault.NoActionBar">
              <activity android:name=".MainActivity" android:exported="true">
                <intent-filter>
                  <action android:name="android.intent.action.MAIN"/>
                  <category android:name="android.intent.category.LAUNCHER"/>
                </intent-filter>
              </activity>
              <service android:name=".JarvisService" android:exported="false" android:foregroundServiceType="microphone"/>
            </application>
          </manifest>
          EOF
          cat > app/src/main/java/com/jarvis/app/MainActivity.kt <<'EOF'
          package com.jarvis.app
          import android.app.Activity
          import android.content.Intent
          import android.net.Uri
          import android.os.Bundle
          import android.provider.Settings
          import android.view.View
          import android.widget.*

          object Bus {
            val lines = StringBuilder()
            var on: (() -> Unit)? = null
            fun add(s: String) { lines.append(s).append("\n\n"); on?.invoke() }
          }

          class MainActivity : Activity() {
            override fun onCreate(s: Bundle?) {
              super.onCreate(s)
              val col = LinearLayout(this)
              col.orientation = LinearLayout.VERTICAL
              col.setPadding(40, 120, 40, 40)
              val title = TextView(this)
              title.text = "Jarvis"
              title.textSize = 28f
              col.addView(title)
              fun btn(t: String, f: () -> Unit) {
                val b = Button(this)
                b.text = t
                b.setOnClickListener { f() }
                col.addView(b)
              }
              btn("1. Permissions do") {
                requestPermissions(arrayOf("android.permission.RECORD_AUDIO", "android.permission.READ_CONTACTS", "android.permission.CALL_PHONE", "android.permission.POST_NOTIFICATIONS"), 1)
              }
              btn("2. Display over other apps allow karo") {
                startActivity(Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION, Uri.parse("package:$packageName")))
              }
              btn("3. Jarvis chalu karo") { startForegroundService(Intent(this, JarvisService::class.java)) }
              btn("Band karo") { stopService(Intent(this, JarvisService::class.java)) }
              val log = TextView(this)
              val sv = ScrollView(this)
              sv.addView(log)
              col.addView(sv, LinearLayout.LayoutParams(-1, 0, 1f))
              val row = LinearLayout(this)
              val et = EditText(this)
              et.hint = "Command likho"
              row.addView(et, LinearLayout.LayoutParams(0, -2, 1f))
              val go = Button(this)
              go.text = "Bhejo"
              go.setOnClickListener {
                val x = et.text.toString()
                et.setText("")
                Bus.add("Aap: $x")
                Bus.add("Jarvis: " + Commands.run(this, x))
              }
              row.addView(go)
              col.addView(row)
              setContentView(col)
              Bus.on = { runOnUiThread { log.text = Bus.lines; sv.fullScroll(View.FOCUS_DOWN) } }
              log.text = Bus.lines
            }
          }
          EOF
          cat > app/src/main/java/com/jarvis/app/JarvisService.kt <<'EOF'
          package com.jarvis.app
          import android.app.*
          import android.content.Intent
          import android.os.*
          import android.speech.*
          import android.speech.tts.TextToSpeech
          import java.util.Locale

          class JarvisService : Service() {
            private var sr: SpeechRecognizer? = null
            private var tts: TextToSpeech? = null
            private val h = Handler(Looper.getMainLooper())
            private var running = false
            private var awaiting = false

            override fun onBind(i: Intent?): IBinder? = null

            override fun onStartCommand(i: Intent?, f: Int, id: Int): Int {
              val nm = getSystemService(NotificationManager::class.java)
              nm.createNotificationChannel(NotificationChannel("jarvis", "Jarvis", NotificationManager.IMPORTANCE_LOW))
              val n = Notification.Builder(this, "jarvis")
                .setContentTitle("Jarvis sun raha hai")
                .setContentText("Bolo: Hey Jarvis, Rahul ko call karo")
                .setSmallIcon(android.R.drawable.ic_btn_speak_now).build()
              startForeground(1, n)
              if (tts == null) tts = TextToSpeech(this) { st -> if (st == TextToSpeech.SUCCESS) tts?.language = Locale("en", "IN") }
              running = true
              listen()
              return START_STICKY
            }

            private val ls = object : RecognitionListener {
              override fun onReadyForSpeech(p: Bundle?) {}
              override fun onBeginningOfSpeech() {}
              override fun onRmsChanged(v: Float) {}
              override fun onBufferReceived(b: ByteArray?) {}
              override fun onEndOfSpeech() {}
              override fun onError(e: Int) { again() }
              override fun onResults(r: Bundle?) {
                r?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)?.firstOrNull()?.let { handle(it) }
                again()
              }
              override fun onPartialResults(p: Bundle?) {}
              override fun onEvent(t: Int, p: Bundle?) {}
            }

            private fun again() { h.postDelayed({ listen() }, 600) }

            private fun listen() {
              if (!running) return
              if (sr == null) {
                sr = SpeechRecognizer.createSpeechRecognizer(this)
                sr?.setRecognitionListener(ls)
              }
              sr?.cancel()
              val i = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH)
                .putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
                .putExtra(RecognizerIntent.EXTRA_LANGUAGE, "en-IN")
              sr?.startListening(i)
            }

            private fun handle(text: String) {
              val l = text.lowercase()
              val m = Regex("(jarvis|jarwis|jervis|garvis)").find(l)
              if (m != null) {
                val cmd = l.substring(m.range.last + 1).trim()
                if (cmd.isEmpty()) { awaiting = true; say("Ji, boliye") } else exec(cmd)
              } else if (awaiting) {
                awaiting = false
                exec(l)
              }
            }

            private fun exec(cmd: String) {
              Bus.add("Aap: $cmd")
              val r = Commands.run(this, cmd)
              Bus.add("Jarvis: $r")
              say(r)
            }

            private fun say(t: String) { tts?.speak(t, TextToSpeech.QUEUE_FLUSH, null, "j") }

            override fun onDestroy() {
              running = false
              h.removeCallbacksAndMessages(null)
              sr?.destroy()
              tts?.shutdown()
              super.onDestroy()
            }
          }
          EOF
          cat > app/src/main/java/com/jarvis/app/Commands.kt <<'EOF'
          package com.jarvis.app
          import android.content.Context
          import android.content.Intent
          import android.content.pm.PackageManager
          import android.net.Uri
          import android.provider.AlarmClock
          import android.provider.ContactsContract

          object Commands {
            private fun go(c: Context, i: Intent) { i.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK); c.startActivity(i) }

            private fun number(c: Context, name: String): String? {
              if (c.checkSelfPermission("android.permission.READ_CONTACTS") != PackageManager.PERMISSION_GRANTED) return null
              val p = ContactsContract.CommonDataKinds.Phone.CONTENT_URI
              c.contentResolver.query(p, arrayOf(ContactsContract.CommonDataKinds.Phone.NUMBER),
                ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME + " LIKE ?", arrayOf("%" + name + "%"), null)?.use { q ->
                if (q.moveToFirst()) return q.getString(0).replace(Regex("[^\\d+]"), "")
              }
              return null
            }

            private fun intl(n: String): String { val d = n.replace("+", ""); return if (d.length == 10) "91" + d else d }

            fun run(c: Context, raw: String): String {
              var t = raw.trim().lowercase()
                .replace(Regex("\\b(karo|kardo|kar do|lagao|please|zara|ko|ke liye)\\b"), " ")
                .replace(Regex("\\s+"), " ").trim()
              Regex("^(\\S+) (call|phone|whatsapp|sms|message)\\b(.*)$").find(t)?.let { t = it.groupValues[2] + " " + it.groupValues[1] + it.groupValues[3] }
              Regex("^(.+) (open|kholo|khol)$").find(t)?.let { t = "open " + it.groupValues[1] }
              try {
                Regex("^(call|phone) (.+)$").find(t)?.let { m ->
                  val who = m.groupValues[2]
                  val n = number(c, who) ?: return "$who ka number nahi mila. Contacts permission check karo."
                  val ok = c.checkSelfPermission("android.permission.CALL_PHONE") == PackageManager.PERMISSION_GRANTED
                  go(c, Intent(if (ok) Intent.ACTION_CALL else Intent.ACTION_DIAL, Uri.parse("tel:" + n)))
                  return "$who ko call laga raha hoon."
                }
                Regex("^(whatsapp|sms|message) (\\S+)\\s*(.*)$").find(t)?.let { m ->
                  val who = m.groupValues[2]
                  val msg = m.groupValues[3].removePrefix("ki ").removePrefix("ke")
                  val n = number(c, who) ?: return "$who ka number nahi mila."
                  val u = if (m.groupValues[1] == "whatsapp") "https://wa.me/" + intl(n) + "?text=" + Uri.encode(msg) else "sms:" + n + "?body=" + Uri.encode(msg)
                  go(c, Intent(Intent.ACTION_VIEW, Uri.parse(u)))
                  return "$who ke liye message taiyaar hai. Send aap dabana."
                }
                Regex("^timer (\\d+) ?(\\w*)").find(t)?.let { m ->
                  val n = m.groupValues[1].toInt()
                  val u = m.groupValues[2]
                  val s = if (u.startsWith("sec")) n else if (u.startsWith("h") || u.startsWith("g")) n * 3600 else n * 60
                  go(c, Intent(AlarmClock.ACTION_SET_TIMER).putExtra(AlarmClock.EXTRA_LENGTH, s).putExtra(AlarmClock.EXTRA_SKIP_UI, true))
                  return "Timer laga diya."
                }
                Regex("^alarm (\\d{1,2})(?:[: ](\\d{2}))?").find(t)?.let { m ->
                  var hr = m.groupValues[1].toInt()
                  val mi = m.groupValues[2].ifEmpty { "0" }.toInt()
                  if (t.contains("pm") && hr < 12) hr += 12
                  go(c, Intent(AlarmClock.ACTION_SET_ALARM).putExtra(AlarmClock.EXTRA_HOUR, hr).putExtra(AlarmClock.EXTRA_MINUTES, mi).putExtra(AlarmClock.EXTRA_SKIP_UI, true))
                  return "Alarm laga diya."
                }
                Regex("^(open|kholo|khol) (.+)$").find(t)?.let { m ->
                  val q = m.groupValues[2]
                  val pm = c.packageManager
                  val apps = pm.queryIntentActivities(Intent(Intent.ACTION_MAIN).addCategory(Intent.CATEGORY_LAUNCHER), 0)
                  val hit = apps.firstOrNull { it.loadLabel(pm).toString().lowercase().contains(q) }
                  val li = hit?.let { pm.getLaunchIntentForPackage(it.activityInfo.packageName) }
                  if (li != null) { go(c, li); return q + " khol raha hoon." }
                  return q + " naam ka app nahi mila."
                }
                Regex("^(navigate|directions|map|route|rasta)( to)? (.+)$").find(t)?.let { m ->
                  go(c, Intent(Intent.ACTION_VIEW, Uri.parse("geo:0,0?q=" + Uri.encode(m.groupValues[3]))))
                  return "Map khol raha hoon."
                }
                Regex("^youtube (.+)$").find(t)?.let { m ->
                  go(c, Intent(Intent.ACTION_VIEW, Uri.parse("https://www.youtube.com/results?search_query=" + Uri.encode(m.groupValues[1]))))
                  return "YouTube par dhoondh raha hoon."
                }
                go(c, Intent(Intent.ACTION_VIEW, Uri.parse("https://www.google.com/search?q=" + Uri.encode(raw))))
                return "Google par dhoondh raha hoon."
              } catch (e: Exception) {
                return "Command chalane mein dikkat: " + e.message
              }
            }
          }
          EOF
          gradle assembleDebug --no-daemon
      - uses: actions/upload-artifact@v4
        with:
          name: Jarvis-APK
          path: jv/app/build/outputs/apk/debug/app-debug.apk
