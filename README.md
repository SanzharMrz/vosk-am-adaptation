# vosk-am-adaptation

Fine-Tuning русскоязычной акустической модели [http://vosk-model-ru-0.10.zip](https://alphacephei.com/vosk/models/vosk-model-ru-0.10.zip)

# Prepare data
Официальный гайд с Kaldi [Data Preparation](https://kaldi-asr.org/doc/data_prep.html)

В моем кейсе, когда каждый новый wav файл, это новая запись, и плюс отсутсвуют segments, все файлики, где имеется recording-id могут быть заменены на utterance-id
>  if the "segments" file does not exist, the first token on each line of "wav.scp" file is just the utterance id. "
 
![wav.scp](https://user-images.githubusercontent.com/48170101/117793265-e0586180-b26d-11eb-96d3-4614ed6c66c7.png)
