```python
import subprocess
def extract_audio(input_video_path, out_audio_path):
	"""使用ffmpeg提取视频中的音频并转换为单声道16k采样率的ogg格式"""
	
	subprocess.run([
		"ffmpeg", "-i", input_video_path,
		"-ac", "1",
		"-ar", "16000",
		"-vn",
		"-c:a", "libopus",
		"-b:a", "16k", # 更小的码率
		"-y",
		out_audio_path
		], check=True)
	
	return out_audio_path
```