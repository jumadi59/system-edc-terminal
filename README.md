## Overview
Library support beberapa type EDC untuk mempermudah devlop aplikasi di semua type EDC dan sudah support beberapa pembayaran seperti Credit/Debit dan beberapa prepaid seperti BRIZZI, TAPCASH, FLAZZ dan E-MONEY

Support Feature:
- NFC
- Printer
- Led
- SAM
- EMV Credit/Debit

Requirment:
- Minimum Android SDK 21

### Android Library
- core.aar
- prepaid-lib.aar
- device-{name}.aar

Add Implementation in Gradle
```gradle
dependencies {
    implementation 'androidx.preference:preference-ktx:1.2.0'
    implementation group: 'org.bouncycastle', name: 'bcprov-jdk14', version: '1.70'
    implementation files('libs/core.aar')
    implementation files('libs/prepaid-lib.aar')
    implementation files('libs/device-{name}.aar')
}
```

      
#### Funcition Spesification

Intitalize
```kotlin
DeviceHelper.getInstance().connect(this) {
            if (it) {
                DeviceManager.setDeviceHelper(DeviceHelper.deviceHelper)
                Toast.makeText(this, "Hardware Device connect", Toast.LENGTH_LONG).show()
            } else {
                Toast.makeText(this, "Hardware Device no connect", Toast.LENGTH_LONG).show()
            }
        }
```

Reader NFC
```kotlin
RFCardManager.getInstance().start( object : OnReadListener<String> {
            override suspend fun onRead(rfCardHelper: BaseRFCardHelper): BalanceResult {
                val response = rfCardHelper.transmit(byteArrayOf(0x00, 0x01))
                
                return prepaidHelperImpl.checkBalance()
            }

            override fun onResult(result: String?) {
                RFCardManager.getInstance().close()
            }

            override fun onError(e: ManagerException?) {
                  Toast.makeText(this, e?.message?:"Read Card Error! Card not recognized", Toast.LENGTH_LONG).show()
            }
        })
```

