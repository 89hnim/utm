# Gắn tracking utm
https://developers.google.com/analytics/devguides/collection/android/v4/campaigns

Thêm dependency trong build.gradle(app)
```
  implementation 'com.android.installreferrer:installreferrer:1.1.2'
  implementation "com.google.android.gms:play-services-analytics:17.0.0"
```

Thêm vào project-level build.gradle (nếu chưa có)
``` 
classpath 'com.google.gms:google-services:3.0.0'
```

Copy code dưới vào 1 class mới
```
const val prefKey = "checkedInstallReferrer"

fun MainActivity.initCampaign() {
    checkInstallReferrer()
}

private fun MainActivity.checkInstallReferrer() {
    if (getPreferences(MODE_PRIVATE).getBoolean(prefKey, false)) {
        return
    }
    val referrerClient = InstallReferrerClient.newBuilder(this).build()
    backgroundExecutor.execute { getInstallReferrerFromClient(referrerClient) }
}

private fun MainActivity.getInstallReferrerFromClient(referrerClient: InstallReferrerClient) {
    referrerClient.startConnection(object : InstallReferrerStateListener {
        override fun onInstallReferrerSetupFinished(responseCode: Int) {
            when (responseCode) {
                InstallReferrerResponse.OK -> {
                    val response: ReferrerDetails?
                    response = try {
                        referrerClient.installReferrer
                    } catch (e: RemoteException) {
                        e.printStackTrace()
                        return
                    }
                    val referrerUrl = response.installReferrer
                    Log.d("AppDebug", " Url $referrerUrl")

                    trackInstallReferrer(referrerUrl)

                    // Only check this once.
                    getPreferences(MODE_PRIVATE).edit().putBoolean(prefKey, true)
                        .apply()

                    // End the connection
                    referrerClient.endConnection()
                }
                InstallReferrerResponse.FEATURE_NOT_SUPPORTED -> {
                }
                InstallReferrerResponse.SERVICE_UNAVAILABLE -> {
                }
            }
        }

        override fun onInstallReferrerServiceDisconnected() {}
    })
}

// Tracker for Classic GA (call this if you are using Classic GA only)
private fun MainActivity.trackInstallReferrer(referrerUrl: String) {
    Handler(mainLooper).post {
        val receiver = CampaignTrackingReceiver()
        val intent = Intent("com.android.vending.INSTALL_REFERRER")
        intent.putExtra("referrer", referrerUrl)
        receiver.onReceive(applicationContext, intent)
    }
}
```

Sử dụng: trong MainActivity
```
 val backgroundExecutor: Executor = Executors.newSingleThreadExecutor()
override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        initCampaign()
        ...
}
```
