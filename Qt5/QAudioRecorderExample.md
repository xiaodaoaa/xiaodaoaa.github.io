要使用`QAudioRecorder`录制音频，可以按照以下步骤操作。这些步骤假设你使用的是Qt框架的C++版本。

1. **创建并配置`QAudioRecorder`对象：**

   ```cpp
   #include <QAudioRecorder>
   #include <QAudioEncoderSettings>
   #include <QUrl>

   QAudioRecorder* audioRecorder = new QAudioRecorder(this);

   // 配置录音设置
   QAudioEncoderSettings audioSettings;
   audioSettings.setCodec("audio/pcm");
   audioSettings.setSampleRate(44100);
   audioSettings.setBitRate(128000);
   audioSettings.setChannelCount(2);
   audioSettings.setQuality(QMultimedia::HighQuality);
   audioSettings.setEncodingMode(QMultimedia::ConstantBitRateEncoding);

   audioRecorder->setEncodingSettings(audioSettings);

   // 设置输出位置
   audioRecorder->setOutputLocation(QUrl::fromLocalFile("output.wav"));
   ```

2. **开始录音：**

   ```cpp
   audioRecorder->record();
   ```

3. **停止录音：**

   ```cpp
   audioRecorder->stop();
   ```

4. **处理录音状态的变化（可选）：**

   你可以连接到`QAudioRecorder`的信号以处理录音状态的变化，例如：

   ```cpp
   connect(audioRecorder, &QAudioRecorder::stateChanged, this, [](QMediaRecorder::State state){
       if (state == QMediaRecorder::RecordingState) {
           qDebug() << "Recording started...";
       } else if (state == QMediaRecorder::StoppedState) {
           qDebug() << "Recording stopped.";
       }
   });
   ```

完整示例代码如下：

```cpp
#include <QCoreApplication>
#include <QAudioRecorder>
#include <QAudioEncoderSettings>
#include <QUrl>
#include <QObject>
#include <QDebug>
#include <QTimer>

int main(int argc, char *argv[])
{
	QCoreApplication a(argc, argv);

	QAudioRecorder* audioRecorder = new QAudioRecorder(&a);

	// 设置音频输入（如果有多设备）
	QStringList inputs = audioRecorder->audioInputs();
	if (!inputs.isEmpty()) {
		audioRecorder->setAudioInput(inputs.first());
	}

	QAudioEncoderSettings audioSettings;
	audioSettings.setCodec("audio/pcm");
	audioSettings.setSampleRate(44100);
	audioSettings.setBitRate(128000);
	audioSettings.setChannelCount(2);
	audioSettings.setQuality(QMultimedia::HighQuality);
	audioSettings.setEncodingMode(QMultimedia::ConstantBitRateEncoding);

	audioRecorder->setEncodingSettings(audioSettings);
	audioRecorder->setOutputLocation(QUrl::fromLocalFile("output.wav"));

	QObject::connect(audioRecorder, &QAudioRecorder::stateChanged, [](QMediaRecorder::State state) {
		if (state == QMediaRecorder::RecordingState) {
			qDebug() << "Recording started...";
		}
		else if (state == QMediaRecorder::StoppedState) {
			qDebug() << "Recording stopped.";
		}
	});

	audioRecorder->record();

	// 录制10秒
	QTimer::singleShot(10000, audioRecorder, &QAudioRecorder::stop);

	return a.exec();
}
```

这个示例将在启动后开始录制音频，并在10秒后停止录制。你可以根据需要调整录制时间和其他设置。