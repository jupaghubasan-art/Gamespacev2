package com.aerox.gamespace

import android.app.Activity
import android.os.Bundle
import android.content.pm.PackageManager
import android.graphics.Color
import android.view.Gravity
import android.widget.Button
import android.widget.LinearLayout
import android.widget.ScrollView
import android.widget.TextView
import rikka.shizuku.Shizuku
import java.io.BufferedReader
import java.io.InputStreamReader

class MainActivity : Activity() {

private lateinit var status: TextView
private lateinit var output: TextView

private val permissionRequestCode = 4107
private val mlbbPackage = "com.mobile.legends"

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)

    window.statusBarColor = Color.rgb(8, 12, 24)
    window.navigationBarColor = Color.rgb(8, 12, 24)

    val layout = LinearLayout(this).apply {
        orientation = LinearLayout.VERTICAL
        setPadding(24, 32, 24, 24)
        setBackgroundColor(Color.rgb(8, 12, 24))
    }

    val title = TextView(this).apply {
        text = "GAME SPACE V2"
        textSize = 26f
        gravity = Gravity.CENTER
        setTextColor(Color.WHITE)
    }
    layout.addView(title)

    status = TextView(this).apply {
        text = "MLBB PERFORMANCE ENGINE"
        textSize = 15f
        setTextColor(Color.CYAN)
        setPadding(0, 24, 0, 16)
    }
    layout.addView(status)

    layout.addView(makeButton("Request Shizuku Permission") {
        requestPermission()
    })

    layout.addView(makeButton("Check Performance Settings") {
        runCheck()
    })

    layout.addView(makeButton("Try 90 Hz") {
        runCommand(
            "settings put system peak_refresh_rate 90.0; " +
            "settings get system peak_refresh_rate"
        )
    })

    layout.addView(makeButton("Try 120 Hz") {
        runCommand(
            "settings put system peak_refresh_rate 120.0; " +
            "settings get system peak_refresh_rate"
        )
    })

    layout.addView(makeButton("Reset Refresh Rate Override") {
        runCommand(
            "settings delete system peak_refresh_rate; " +
            "settings delete system min_refresh_rate"
        )
    })

    layout.addView(makeButton("Open MLBB") {
        val intent = packageManager
            .getLaunchIntentForPackage(mlbbPackage)

        if (intent != null) {
            startActivity(intent)
        } else {
            status.text = "MLBB was not found."
        }
    })

    output = TextView(this).apply {
        text = "Results will appear here."
        textSize = 13f
        setTextColor(Color.WHITE)
        setPadding(0, 24, 0, 16)
    }
    layout.addView(output)

    val scroll = ScrollView(this)
    scroll.addView(layout)
    setContentView(scroll)
}

private fun makeButton(
    label: String,
    action: () -> Unit
): Button {
    return Button(this).apply {
        text = label
        isAllCaps = false
        setOnClickListener { action() }
    }
}

private fun requestPermission() {
    if (!Shizuku.pingBinder()) {
        status.text = "Start Shizuku first."
        return
    }

    if (Shizuku.checkSelfPermission() ==
        PackageManager.PERMISSION_GRANTED
    ) {
        status.text = "Shizuku permission granted."
    } else {
        Shizuku.requestPermission(permissionRequestCode)
        status.text = "Permission request sent."
    }
}

private fun runCheck() {
    if (!Shizuku.pingBinder()) {
        status.text = "Shizuku is not running."
        return
    }

    if (Shizuku.checkSelfPermission() !=
        PackageManager.PERMISSION_GRANTED
    ) {
        status.text = "Grant Shizuku permission first."
        return
    }

    runCommand(
        "settings get system peak_refresh_rate; " +
        "settings get system min_refresh_rate; " +
        "cmd game list"
    )
}

private fun runCommand(command: String) {
    if (!Shizuku.pingBinder() ||
        Shizuku.checkSelfPermission() !=
        PackageManager.PERMISSION_GRANTED
    ) {
        status.text = "Shizuku permission required."
        return
    }

    Thread {
        val result = try {
            val process = Shizuku.newProcess(
                arrayOf("sh", "-c", command),
                null,
                null
            )

            val stdout = BufferedReader(
                InputStreamReader(process.inputStream)
            ).readText()

            val stderr = BufferedReader(
                InputStreamReader(process.errorStream)
            ).readText()

            val exitCode = process.waitFor()

            "Exit code: $exitCode\n" +
            "Output:\n$stdout\nErrors:\n$stderr"
        } catch (e: Exception) {
            "Command failed: ${e.message}"
        }

        runOnUiThread {
            output.text = result
            status.text = "Command completed. Verify actual results."
        }
    }.start()
}

}
