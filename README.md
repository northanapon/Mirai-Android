[![](https://jitpack.io/v/InDistinct-Studio/Mirai-Android.svg)](https://jitpack.io/#InDistinct-Studio/Mirai-Android)


# Mirai OCR Library

## Prerequisite

- Android SDK Version 21 or Higher
- Google Play Services
- `Internet` and `Camera` Permissions in Manfiest

## Installation Guide

- Add this to your `build.gradle` at the root of the project

  ```
	allprojects {
		repositories {
			...
			maven { url 'https://jitpack.io' }
		}
	}
  ```
- Add your dependency in your app or feature module as below
```
	dependencies {
		implementation 'com.github.Seen-Digital:mirai-android:<version>'
	}
```

- Sync your gradle project and Mirai is now available for your application

## HOW-TO

1. Initialize SDK by calling `init` function with your API Key. Also, you will also need to implement `Mirai.OnInitializedListener` to check whether the initialization is completed or not

    ```kotlin
    Mirai.init(this, "API_KEY", listener)
    ```


1. Create the `CardImage` object which will be used as an input for OCR

    ```kotlin
    val card = CardImage(this.image!!, imageInfo.rotationDegrees)
    ```

1. Call the `scanIDCard` method from `Mirai` object class which will extract card information from the image. There are 3 required parameters.
    - `card`: Input card image
    - `resultListener`: The listener to receive recognition result. This will provide the information of cards that can be detected in `IDCardResult` object

      ```kotlin
      imageAnalyzer.setAnalyzer(cameraExecutor) { imageProxy ->
              imageProxy.run {
                val card = CardImage(this.image!!, imageInfo.rotationDegrees)
                Mirai.scanIDCard(card) {result ->
                  this@MainActivity.displayResult(result)
                  imageProxy.close()
                }
              }
            }
       ```

1. Use the resulting `IDCardResult` object will consist of the following fields:
    - `error`: If the recognition is successful, the `error` will be null. In case of unsuccessful scan, the `error.errorMessage` will contain the problem of the recognition.
    - `isFrontSide`: A boolean flag indicates whether the scan found the front side (`true`) or back side (`false`) of the card ID.
    - `confidence`: A value between 0.0 to 1.0 (higher values mean more likely to be an ID card).
    - `isFrontCardFull`: A boolean flag indicates whether the scan detect a full front-sde card.
    - `texts`: A list of OCR results. An OCR result consists of `type` and `text`.
        - `type`: Type of information. Right now, Mirai support 3 types: `ID`, `SERIAL_NUMBER`, and `LASER_CODE`
        - `text`: OCR text based on the `type`.
    - `fullImage`: A bitmap image of the full frame used during scanning.
    - `croppedImage`: A bitmap image of the card. This is available if `isFrontSide` is `true`.
    - `faceOnCardImage`: A bitmap image of the face on the card. This is available if `isFrontSide` is `true`.
    - `classificationResult`: A result from ML image labeling, available if `isFrontCardFull` is `true`.
	  - `confidence`: A confidence value from 0 to 1. Higher values mean the images are more likely to be good quality. The threshold of `0.6` to `0.9` is recommended. 
	  - `error`: An object for error messages.

    ```kotlin
    private fun displayResult(result: IDCardResult) {
        result.run {
            // Indicator for front side of the card (true means front).
            val isFrontSide = isFrontSide
            // Indicator for front side of the card and fully capture (true means front and fuull).
            val isFrontCardFull = isFrontCardFull
            // fullImage is always available.
            val capturedImage = fullImage
            // cropped image is only available for front side scan result.
            val cardImage = croppedImage
            // face image (right-bottom corner is only available for front size scan result.
            val faceOnCardImage = faceImage
            // confidence 1: readable information on card
            val confidenceByInfo = confidence
            // confidence 2: ML-based confidence
            val confidenceByML = classificationResult?.confidence
            // text data
            val textData = texts
            // error
            val error = error
        }
    }
      ```

### Example Code

```kotlin
package work.indistinct.mirai.demo

import work.indistinct.mirai.CardImage
import work.indistinct.mirai.IDCardResult
import work.indistinct.mirai.Mirai
import android.Manifest
import android.content.pm.PackageManager
import android.graphics.Rect
import android.os.Build
import android.os.Bundle
import android.widget.Button
import android.widget.Switch
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity
import androidx.camera.core.*
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.camera.view.PreviewView
import androidx.core.content.ContextCompat
import android.widget.Toast
import kotlinx.android.synthetic.main.activity_main.*
import java.util.concurrent.ExecutorService
import java.util.concurrent.Executors
import kotlin.math.abs
import kotlin.math.max
import kotlin.math.min

@androidx.camera.core.ExperimentalGetImage
class MainActivity : AppCompatActivity(), Mirai.OnInitializedListener {

    lateinit var swapCameraButton: Button
    lateinit var resultText: TextView
    lateinit var confidenceText: TextView
    lateinit var faceText: TextView
    lateinit var faceDetectionSwitch: Switch

    private var preview: Preview? = null
    private var imageCapture: ImageCapture? = null
    private var imageAnalyzer: ImageAnalysis? = null
    private var camera: Camera? = null
    private var lensFacing: Int = CameraSelector.LENS_FACING_BACK
    private var cameraProvider: ProcessCameraProvider? = null

    private var previewWidth: Int = 1
    private var previewHeight: Int = 1
    private var imgProxyWidth: Int = 1
    private var imgProxyHeight: Int = 1
    private var imgProxyRotationDegree: Int = 0
    private lateinit var cameraExecutor: ExecutorService

    companion object {
        private const val RATIO_4_3_VALUE = 4.0 / 3.0
        private const val RATIO_16_9_VALUE = 16.0 / 9.0
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        Mirai.init(this,"ajMbRHTFPtUo9RzpSAMd", this)

        resultText = findViewById(R.id.resultTextView)
        confidenceText = findViewById(R.id.confidenceTextView)
        swapCameraButton = findViewById(R.id.swapCameraButton)
        faceText = findViewById(R.id.faceTextView)
        faceDetectionSwitch = findViewById(R.id.faceDetectionSwitch)
        swapCameraButton.setOnClickListener {
            swapCamera()
        }

        if (ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA) != PackageManager.PERMISSION_GRANTED) {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                requestPermissions(arrayOf(Manifest.permission.CAMERA), 1000)
            }
        }

    }

    override fun onCompleted() {
        Toast.makeText(this, "Init Completed", Toast.LENGTH_LONG).show()
        cameraExecutor = Executors.newSingleThreadExecutor()
        setupCamera()
    }

    override fun onError(message: String) {
        Toast.makeText(this, message, Toast.LENGTH_LONG).show()
    }

    private fun setupCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)
        cameraProviderFuture.addListener({
            // CameraProvider
            cameraProvider = cameraProviderFuture.get()

            // Build and bind the camera use cases
            bindCameraUseCases()
        }, ContextCompat.getMainExecutor(this))
    }

    private fun bindCameraUseCases() {
        val cameraSelector = CameraSelector.Builder().requireLensFacing(lensFacing).build()

//        val metrics = DisplayMetrics().also { previewView.display.getRealMetrics(it) }

//        val screenAspectRatio = aspectRatio(metrics.widthPixels, metrics.heightPixels)
        previewView.scaleType = PreviewView.ScaleType.FIT_START
        val rotation = previewView.display.rotation

//        val previewChild = previewView.getChildAt(0)
        previewWidth = (previewView.width * previewView.scaleX).toInt()
        previewHeight = (previewView.height * previewView.scaleY).toInt()
        val screenAspectRatio = aspectRatio(previewWidth, previewHeight)

        preview = Preview.Builder()
            .setTargetRotation(rotation)
            .setTargetAspectRatio(screenAspectRatio)
            .build()


        imageCapture = ImageCapture.Builder()
            .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
            .setTargetAspectRatio(screenAspectRatio)
            .setTargetRotation(rotation)
            .build()

        imageAnalyzer = ImageAnalysis.Builder()
            .setTargetAspectRatio(screenAspectRatio)
            .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
            .build()
            .also {
                it.setAnalyzer(cameraExecutor) { imageProxy ->

                    imageProxy.run {
                        val reverseDimens = imageInfo.rotationDegrees == 90 || imageInfo.rotationDegrees == 270
                        imgProxyWidth = if (reverseDimens) imageProxy.height else imageProxy.width
                        imgProxyHeight = if (reverseDimens) imageProxy.width else imageProxy.height
                        imgProxyRotationDegree = imageInfo.rotationDegrees
                        val card = CardImage(this.image!!, imageInfo.rotationDegrees)
                        Mirai.scanIDCard(card) { result ->
                            this@MainActivity.displayResult(result)
                            imageProxy.close()
//                            if (faceDetectionSwitch.isChecked) {
//                                Mirai.checkFace(result) { faceResult ->
//                                    this@MainActivity.displayResult(result, faceResult)
//                                    imageProxy.close()
//                                }
//                            } else {
//                                this@MainActivity.displayResult(result, null)
//                                imageProxy.close()
//                            }
                        }
                    }
                }
            }

        cameraProvider?.unbindAll()

        try {
            // A variable number of use-cases can be passed here -
            // camera provides access to CameraControl & CameraInfo
            camera = cameraProvider?.bindToLifecycle(
                this, cameraSelector, preview, imageAnalyzer)

            // Attach the viewfinder's surface provider to preview use case
            preview?.setSurfaceProvider(previewView.surfaceProvider)
        } catch (exc: Exception) {
        }
    }

    private fun swapCamera() {
        cameraProvider?.unbindAll()
        lensFacing = if (lensFacing == CameraSelector.LENS_FACING_BACK) {
            CameraSelector.LENS_FACING_FRONT
        } else {
            CameraSelector.LENS_FACING_BACK
        }
        bindCameraUseCases()
    }

    private fun aspectRatio(width: Int, height: Int): Int {
        val previewRatio = max(width, height).toDouble() / min(width, height)
        if (abs(previewRatio - RATIO_4_3_VALUE) <= abs(previewRatio - RATIO_16_9_VALUE)) {
            return AspectRatio.RATIO_4_3
        }
        return AspectRatio.RATIO_16_9
    }

    private fun displayResult(result: IDCardResult) {
        result.run {
            // Indicator for front side of the card (true means front).
            val isFrontSide = isFrontSide
            // Indicator for front side of the card and fully capture (true means front and fuull).
            val isFrontCardFull = isFrontCardFull
            // fullImage is always available.
            val capturedImage = fullImage
            // cropped image is only available for front side scan result.
            val cardImage = croppedImage
            // face image (right-bottom corner is only available for front size scan result.
            val faceOnCardImage = faceImage
            // confidence 1: readable information on card
            val confidenceByInfo = confidence
            // confidence 2: ML-based confidence
            val confidenceByML = classificationResult?.confidence
            // text data
            val textData = texts
            // error
            val error = error

            if (error != null) {
                resultText.text = "ERROR: ${error.errorMessage}"
                return
            }

            if (textData != null) {
                resultText.text = "TEXTS -> ${textData!!.joinToString("\n")}, isFrontside -> $isFrontSide"
            } else {
                resultText.text = "TEXTS -> NULL, isFrontside -> $isFrontSide"
            }

            confidenceText.text = "%.3f ".format(confidenceByInfo)
            if (isFrontSide != null && isFrontSide as Boolean) {
                // cropped image is only available for front side scan result.
                confidenceText.text = "%s, Full: %s".format(confidenceText.text, isFrontCardFull)

                if (classificationResult != null && classificationResult!!.error == null) {
                    confidenceText.text = "%s (%.3f)".format(confidenceText.text, confidenceByML)
                }
            }
        }
    }
```
