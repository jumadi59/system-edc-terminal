## Overview

Library support beberapa type EDC untuk mempermudah devlop aplikasi di semua type EDC dan sudah
support beberapa pembayaran seperti Credit/Debit dan beberapa prepaid seperti BRIZZI, TAPCASH, FLAZZ
dan E-MONEY

Support Feature:

- NFC
- Printer
- Led
- SAM
- EMV Credit/Debit

Requirement:

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

### Sample Code

Initialize

```kotlin
DeviceHelper.getInstance().connect(this) {
    if (it) {
        DeviceManager.setDeviceHelper(DeviceHelper.deviceHelper)
        Log.d("Init", "Hardware Device connect")
    } else {
        Log.d("Init", "Hardware Device no connect")
    }
}
```

Printer

```kotlin
val printtModels = mutableListOf<PrintEntity>()
// fontSize 0 = normal, 1 = small
// align 0 = left, 1 center, 2 = right
printtModels.add(
    PrintEntity.PrintText(
        text = "Termianl EDC",
        fontSize = 30,
        align = 1,
        isBold = true
    )
)
printtModels.add(PrintEntity.feed())
printtModels.add(PrintEntity.doubleLine())
printtModels.add(PrintEntity.PrintImage(image = bitmap, align = 1, width = 350))
printtModels.add(PrintEntity.singleLine())
printtModels.add(PrintEntity.PrintQR(text = "12345678", align = 1))
printtModels.add(PrintEntity.feed())
printtModels.add(PrintEntity.feed())
CoroutineScope(Dispatchers.IO).launch {
    PrinterManager.getInstance().connect().printList(printtModels, {
        Log.d("Printer", "print success")
    }, {
        Log.d("Printer", "print error $it")
    })
}
```

Emv Read Card

```kotlin
val emv = DeviceHelper.getInstance().emvConfiguration
emv.configuration(this) {
    Log.d("EMV", "Please insert/swipe card")
    emvConfiguration.startReadCard(amount, object : TransactionResponse {
        override fun onSearchCard(cardType: Int, trackData: TrackData) {
            Log.d("EMV", "getTrack1Data ${trackData.track1Data}")
            Log.d("EMV", "getTrack1Data ${trackData.track2Data}")
            Log.d("EMV", "getTrack1Data ${trackData.track3Data}")
            if (cardType == 8) {
                Log.d("EMV", "cardType CARD_TYPE_SWIPE")
                startInputPin(amount)
            } else {
                Log.d("EMV", "cardType CARD_TYPE_INSERT")
                Log.d("EMV", "payment success")
            }
        }

        override fun onPinEntry(code: Int) {
            Log.d("EMV", "onPinEntry $code")
            startInputPin(amount)
        }

        override fun onEndProcess(code: Int, message: String?) {
            Log.d("EMV", "onEndProcess code $code message $message")
        }

        override fun onError(code: Int, message: String?) {
            Log.d("EMV", "onError code $code message $message")
        }
    })
}
```

EMV Input PIN

```kotlin
val emv = DeviceHelper.getInstance().emvConfiguration
val pinConfig = PinConfig(mTvDigits, btnBack, btnDelete, btnOk, keyboard)
emv.startPinInput(pinConfig, object : PinInputListener {
    override fun onDisplayPin(pin: String) {
        Log.d("InputPin", " onDisplayPin $pin")
    }

    override fun onSuccess(pinBlock: ByteArray) {
        Log.d("InputPin", "pinBlock ${pinBlock.bytesToHex()}")
    }

    override fun onError(code: Int) {
        Log.d("InputPin", "Input PIN error, code $code")
    }

    override fun onTimeout() {
        Log.d("InputPin", "Input PIN time out")
    }

    override fun onCancel() {
        Log.d("InputPin", "Input PIN cancel")
    }

})
```

### Prepaid

#### Sample Code

```kotlin
RFCardManager.getInstance().start(object : OnReadListener<String> {
    override suspend fun onRead(rfCardHelper: BaseRFCardHelper): BalanceResult {
        Log.d("RFCardManager", "onRead")
        val response = rfCardHelper.transmit(byteArrayOf(0x00, 0x01))
        val code = response.code // is value 0 success
        val sw1sw2 = response.sw1sw2 // is value 9000 success
        val data = response.data
        if (response.isSuccess()) {
            return data.bytesToHex()
        } else ManagerException("error transimtt data")
    }

    override fun onResult(result: String?) {
        RFCardManager.getInstance().close()
        Log.d("RFCardManager", "onResult $result")
    }

    override fun onError(e: ManagerException?) {
        Log.d("RFCardManager", "onError ${e?.message}")
    }
})
```

Prepaid Setting

```kotlin
startActivity(Intent(activity, SettingsActivity::class.java))

//custom setting
val module = PrepaidModule(context)
module.pref.edit().putBoolean("is_sam_device", true).apply()
module.pref.edit().putBoolean("is_auto_prepaid", false).apply()
module.pref.edit().putBoolean("is_sam_cloud", false).apply()
module.pref.edit().putString("host_sam_cloud", "http://localhost:5000").apply()

module.pref.edit().putString("index_{slot}", "bni_tapcash_bin").apply()
module.pref.edit().putString("index_{slot}", "mandiri_emoney_bin").apply()
module.pref.edit().putString("index_{slot}", "bri_brizzi_bin").apply()
module.pref.edit().putString("index_{slot}", "bca_flazz_bin").apply()
```

Prepaid Check Balance

```kotlin
val module = PrepaidModule(context)

override suspend fun onRead(rfCardHelper: BaseRFCardHelper): BalanceResult {
    Log.d("RFCardManager", "onRead")
    val prepaidHelperImpl = PrepaidHelperV2(module, rfCardHelper)
    val aid = prepaidHelperImpl.findAid { mockPrepaidInfo(it) }
    outputView.setOutput("Aid: ${aid?.binName}", 4)
    return prepaidHelperImpl.checkBalance()
}

override fun onResult(result: BalanceResult?) {
    Log.d("RFCardManager", "onResult")
    if (result != null) {
        Log.d("RFCardManager", "Card number    : ${result.binName}")
        Log.d("RFCardManager", "Card number    : ${result.cardNo}")
        Log.d("RFCardManager", "Balance        : ${result.balanceRupiah}")
        Log.d(
            "RFCardManager",
            "Time           : ${RFCardManager.getInstance().processTimeMillis / 1000}s"
        )
    }
    RFCardManager.getInstance().close()
}

```

Prepaid Payment

```kotlin
val module = PrepaidModule(context)
fun mockPrepaidInfo(aidCode: AidCode) = PrepaidInfo(
    id = 0,
    aid = "1122334455",
    binName = aidCode.binName,
    bankName = "",
    bankLogo = null,
    prepaidName = "",
    prepaidLogo = null,
    mid = "000100012001946",
    tid = "12194619"
)

override suspend fun onRead(rfCardHelper: BaseRFCardHelper): PrepaidResult {
    prepaidHelperImpl = PrepaidHelperV2(module, rfCardHelper)
    val aid = prepaidHelperImpl.findAid { mockPrepaidInfo(it) }
    outputView.setOutput("Aid: ${aid?.binName}", 4)
    return prepaidHelperImpl.payment(1, "00000000")
}

override fun onResult(result: PrepaidResult?) {
    Log.d("RFCardManager", "onResult")
    if (result != null) {
        Log.d("RFCardManager", "Card number    : ${result.binName}")
        Log.d("RFCardManager", "Card number    : ${result.cardNo}")
        Log.d("RFCardManager", "Last Balance   : ${result.lastBalance}")
        Log.d("RFCardManager", "Current Balance: ${result.currentBalance}")
        Log.d(
            "RFCardManager",
            "Time           : ${RFCardManager.getInstance().processTimeMillis / 1000}s"
        )
    }
    RFCardManager.getInstance().close()
}
```

Data Card
```kotlin
override fun onResult(result: PrepaidResult?) {
    Log.d("RFCardManager", "onResult")
    RFCardManager.getInstance().close()
    when (result.binName) {
        PrepaidHelperImpl.PREPAID_TAPCASH -> {
            val data = (result as PrepaidResult.TapCashResult).toCardData()
            Log.d("PREPAID_TAPCASH", "data   : $data")
        }
        PrepaidHelperImpl.PREPAID_EMONEY -> {
            binding.logoPrepaid.setImageResource(R.drawable.e_money)
            val data = (result as PrepaidResult.EMoneyResult).toCardData()
            Log.d("PREPAID_EMONEY", "data   : $data")
        }
        PrepaidHelperImpl.PREPAID_BRIZZI -> {
            binding.logoPrepaid.setImageResource(R.drawable.card_brizzi)
            val data = (result as PrepaidResult.BrizziResult).toCardData()
            Log.d("PREPAID_BRIZZI", "data   : $data")
        }
        PrepaidHelperImpl.PREPAID_FLAZZ -> {
            binding.logoPrepaid.setImageResource(R.drawable.card_flazz)
            val data = (result as PrepaidResult.FlazzResult).toCardData()
            Log.d("PREPAID_FLAZZ", "data   : $data")
        }
        else -> {}
    }
}
```

Add Card Prepaid

```kotlin
val module = PrepaidModule(context)
module.addProvideService(TronCardComponent.BIN_NAME, object : TronCashImpl() {
    override fun balance(can: ByteArray): Long = 1000000

    override fun deduct(can: ByteArray, amount: Long): Long = 1000000 - amount
})

fun PrepaidHelperImplV2.addTronCard(rfCardHelper: BaseRFCardHelper): PrepaidHelperImplV2 {
    addComponent(TronCardComponent().apply {
        init(
            TronCashHelper(rfCardHelper),
            module.getProvideService(TronCardComponent.BIN_NAME)!! as TronCashImpl
        )
    })
    return this
}

override suspend fun onRead(rfCardHelper: BaseRFCardHelper): PrepaidResult {
    prepaidHelperImpl = PrepaidHelperV2(module, rfCardHelper).addTronCard(rfCardHelper)
    val aid = prepaidHelperImpl.findAid { mockPrepaidInfo(it) }
    outputView.setOutput("Aid: ${aid?.binName}", 4)
    return prepaidHelperImpl.payment(1, "00000000")
}
```
