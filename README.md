# Vosk-am-adaptation / [DRAFT] + [HELP!]

Fine-Tuning русскоязычной акустической модели [http://vosk-model-ru-0.10.zip](https://alphacephei.com/vosk/models/vosk-model-ru-0.10.zip)
# Installing
Ставим Kaldi. Билдим. Качаем модельку.
```bash
# Copy repo
git clone https://github.com/kaldi-asr/kaldi.git

# Enter tools
cd kaldi/tools/ 

# Make
make 

# Enter source and configure
cd ../src
./configure

# Make
make

# Download and unzip model
cd ../
wget https://alphacephei.com/vosk/models/vosk-model-ru-0.10.zip
unzip vosk-model-ru-0.10.zip
```

# Prepare data
Официальный гайд с Kaldi [Data Preparation](https://kaldi-asr.org/doc/data_prep.html). Есть еще код от JohnDoe, для [RM](https://catalog.ldc.upenn.edu/LDC93S3C) датасета, форкнутый репозиторий Kaldi, в котором [prepare_data.py](https://github.com/JohnDoe02/kaldi/blob/private/egs/rm/s5/local/prepare_data.py) парсящий
некий dataset.tsv с колонками,  __"File"__, __"Length"__, __"Directory"__, __"Recognition"__ , здесь стоит обратить внимание, что __"File"__ - это название файла с полным путем и некоторой метой,  "Length" - может быть заполнен чем угодно, "Directory" - полный путь до папки с файлами,  "Recognition" - текст из wav файлов.

> If the "segments" file does not exist, the first token on each line of "wav.scp" file is just the utterance id."

Стоит обратить внимание на сортировку по utterance-id:

> All of these files should be sorted. If they are not sorted, you will get errors when you run the scripts

Помимо этого, нужно проверить свои аудио через `sox --i filename`, и если у вас wav файлы разрезанные, плюс мультиканальность, то: 

> The recording side is a concept that relates to telephone conversations where there are two channels, and if not, it's probably safe to use "A". 

В моем кейсе, когда каждый новый wav файл, это новая запись, и плюс отсутсвуют segments, все файлики, где имеется recording-id могут быть заменены на utterance-id:

*wav.scp*

![wav.scp](https://user-images.githubusercontent.com/48170101/117793265-e0586180-b26d-11eb-96d3-4614ed6c66c7.png)

*utt2spk*

![utt2spk](https://user-images.githubusercontent.com/48170101/117793486-17c70e00-b26e-11eb-8104-9f13f35ca259.png)

*spk2gender*

![spk2gender](https://user-images.githubusercontent.com/48170101/117793875-7e4c2c00-b26e-11eb-9665-d9a4049fa0c8.png)

# Create features

Для экстракции mel признаков и их нормализации используются steps/ скрипты из рецепта [CVTE finetuning](https://github.com/zhaoyi2/CVTE_chain_model_finetune/tree/master/steps), **ВОЗМОЖНО** стоит использовать steps из kaldi [aishell2](https://github.com/kaldi-asr/kaldi/tree/master/egs/aishell2/s5), может ли это __повлиять__ на подготовку признаков? Что касательно скрипта по finetune, они собраны в один sh [finetune_tdnn_1a.sh](https://github.com/kaldi-asr/kaldi/blob/master/egs/aishell2/s5/local/nnet3/tuning/finetune_tdnn_1a.sh), данный pipeline рекомендован в alphacephei [model adaptation](https://alphacephei.com/vosk/adaptation) 

В репозитории приведен скрипт используемый нами, [ft.sh](ft.sh)
Непосредственно кусок с формированием данных (возможны неточности, строить провалидировать):

```bash
# Подсчет mfcc признаков
steps/make_mfcc.sh \
  --cmd "$train_cmd" --nj $nj \
  ${data_dir} exp/make_mfcc/${data_set} mfcc
# нормализация
steps/compute_cmvn_stats.sh ${data_dir} exp/make_mfcc/${data_set} mfcc || exit 1;

utils/fix_data_dir.sh ${data_dir} || exit 1;
# extract mfcc_hires for AM finetuning
utils/copy_data_dir.sh ${data_dir} ${data_dir}_hires

rm -f ${data_dir}_hires/{cmvn.scp,feats.scp}
steps/make_mfcc.sh \
  --cmd "$train_cmd" --nj $nj --mfcc-config conf/mfcc_hires.conf \
  ${data_dir}_hires exp/make_mfcc/${data_set}_hires mfcc_hires

steps/compute_cmvn_stats.sh ${data_dir}_hires exp/make_mfcc/${data_set}_hires mfcc_hires

# Подсчет ivector
sh steps/online/nnet2/extract_ivectors_online.sh $data_dir ivector exp/nnet3_online/ivectors_test

# Alignment с использованием ivector
# К слову в align.sh присутствует наша модификация на размер beam, с целью поддержания как можно большего числа гипотез
# beam=1000
# retry_beam=10000

sh steps/nnet3/align.sh $data_dir data/lang $ali_dir exp/nnet3/ali

# Перенос подсчитанных ali
cp exp/nnet3/ali/ali.1.gz $ali_dir/ali.1.gz
```

# Training

Далее тренируется копия исходной акустической моделью _input.raw_

```bash
steps/nnet3/train_dnn.py --stage=$train_stage \
  --cmd="$decode_cmd" \
  --feat.cmvn-opts="--norm-means=false --norm-vars=false" \
  --trainer.input-model $dir/input.raw \
  --trainer.num-epochs $num_epochs \
  --trainer.optimization.num-jobs-initial $num_jobs_initial \
  --trainer.optimization.num-jobs-final $num_jobs_final \
  --trainer.optimization.initial-effective-lrate $initial_effective_lrate \
  --trainer.optimization.final-effective-lrate $final_effective_lrate \
  --trainer.optimization.minibatch-size $minibatch_size \
  --feat-dir ${data_dir}_hires \
  --lang data/lang \
  --ali-dir ${ali_dir} \
  --feat.online-ivector-dir exp/nnet3_online/ivectors_test \
  --egs.frames-per-eg 100 \ # может ли здесь быть проблема с объемом frames
  --dir $dir || exit 1;
```
После тренировки в сгенерированной папке для эксприментов, появятся файлы модели, на каждой 100ой итерации, скомбинированная модель по всем итерациям и финальная модель

![ls](https://user-images.githubusercontent.com/48170101/117951384-ef0a4b80-b335-11eb-9f4e-2d2f9883432f.png)

# Inference

Процесс декодинга является камнем преткновения в нашем случае. В vosk-api, имеется скрипт для [inference](https://github.com/alphacep/vosk-api/blob/master/python/test/transcribe_scp.py). Который мы собственно и используем как _py_ команду. 

__Кейс: 1__ Копируем одну из вышеупомянутых .mdl файликов в папку _am_ скачанной vosk-модели. Допустим final.mdl (Что характерно, при обучении frameы-per-eg был равен 100). И получаем:

![inference_final](https://user-images.githubusercontent.com/48170101/117957473-eddc1d00-b33b-11eb-9731-36df023a1219.png)

__Кейс: 2__ Копируем любую другую из моделей. Например combined.mdl:

![inference_combined](https://user-images.githubusercontent.com/48170101/117958178-95f1e600-b33c-11eb-8955-74594648fb51.png)

__Кейс: 3__ 100.mdl:

![100.mdl](https://user-images.githubusercontent.com/48170101/117959014-6becf380-b33d-11eb-9358-ed843393abcd.png)

__Еще немного меты:__

![image](https://user-images.githubusercontent.com/48170101/117958791-30eac000-b33d-11eb-93f6-688b09f6e698.png)


