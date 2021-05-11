# vosk-am-adaptation

Fine-Tuning русскоязычной акустической модели [http://vosk-model-ru-0.10.zip](https://alphacephei.com/vosk/models/vosk-model-ru-0.10.zip)

# Prepare data
Официальный гайд с Kaldi [Data Preparation](https://kaldi-asr.org/doc/data_prep.html)
Есть еще код от JohnDoe, для [RM](https://catalog.ldc.upenn.edu/LDC93S3C) датасета, форкнутый репозиторий Kaldi, в котором [prepare_data.py](https://github.com/JohnDoe02/kaldi/blob/private/egs/rm/s5/local/prepare_data.py) парсящий
некий dataset.tsv с колонками,  __"File"__, __"Length"__, __"Directory"__, __"Recognition"__ , здесь стоит обратить внимание, что __"File"__ - это названием файла с полным путем до него,  "Length" может быть заполнен чем угодно, Directory полный путь до папки с файлами,  "Recognition" текст из wav файлов.

В моем кейсе, когда каждый новый wav файл, это новая запись, и плюс отсутсвуют segments, все файлики, где имеется recording-id могут быть заменены на utterance-id
>  if the "segments" file does not exist, the first token on each line of "wav.scp" file is just the utterance id. "
 
wav.scp

![wav.scp](https://user-images.githubusercontent.com/48170101/117793265-e0586180-b26d-11eb-96d3-4614ed6c66c7.png)

utt2spk

![utt2spk](https://user-images.githubusercontent.com/48170101/117793486-17c70e00-b26e-11eb-8104-9f13f35ca259.png)

spk2gender

![spk2gender](https://user-images.githubusercontent.com/48170101/117793875-7e4c2c00-b26e-11eb-9665-d9a4049fa0c8.png)

