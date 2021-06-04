<p align="center">
<img src="flores_logo.png" width="500">
</p>

--------------------------------------------------------------------------------

# The FLORES-101 Evaluation Benchmark for Low-Resource and Multilingual Machine Translation
`FLORES-101` is a Many-to-Many multilingual translation benchmark dataset for 101 languages. 

* **Paper:** [The FLORES-101 Evaluation Benchmark for Low-Resource and Multilingual Machine Translation](https://ai.facebook.com/research/publications/the-flores-101-evaluation-benchmark-for-low-resource-and-multilingual-machine-translation).

* Download `FLORES-101` [**dataset**](https://dl.fbaipublicfiles.com/flores101/dataset/flores101_dataset.tar.gz).

* Read the [**blogpost**](https://ai.facebook.com/blog/the-flores-101-data-set-helping-build-better-translation-systems-around-the-world) and [**paper**](https://ai.facebook.com/research/publications/the-flores-101-evaluation-benchmark-for-low-resource-and-multilingual-machine-translation).

* Evaluation server: [dynabench](https://dynabench.org/flores), Note: instructions to submit model for evaluations will be added

Looking for FLORESv1, which included Nepali, Sinhala, Pashto, and Khmer? Click [here](floresv1/)

## Abstract
One of the biggest challenges hindering progress in low-resource and multilingual machine translation is the lack of good evaluation benchmarks. 
Current evaluation benchmarks either lack good coverage of low-resource languages, consider only restricted domains, or are low quality because they are constructed using semi-automatic procedures. In this work, we introduce the FLORES evaluation benchmark, consisting of 3001 sentences extracted from English Wikipedia and covering a variety of different topics and domains. These sentences have been translated in 101 languages by professional translators through a carefully controlled process. The resulting dataset enables better assessment of model quality on the long tail of low-resource languages, including the evaluation of many-to-many multilingual translation systems, as all translations are multilingually aligned. By publicly releasing such a high-quality and high-coverage dataset, we hope to foster progress in the machine translation community and beyond.

## Download FLORES-101 Dataset 
The data can be downloaded from: [Here](https://dl.fbaipublicfiles.com/flores101/dataset/flores101_dataset.tar.gz).

## Evaluation 

### SPM-BLEU 
For evaluation, we use SentencePiece BLEU (spBLEU) which uses a SentencePiece (SPM) tokenizer with 256K tokens and then BLEU score is computed on the sentence-piece tokenized text. This requires installing sacrebleu using a specific branch:
```bash
git clone --single-branch --branch adding_spm_tokenized_bleu https://github.com/ngoyal2707/sacrebleu.git
cd sacrebleu
python setup.py install
```

### Offline Evaluation

#### Download FLORES-101 dev and devtest dataset

```bash
cd ~/
wget https://dl.fbaipublicfiles.com/flores101/dataset/flores101_dataset.tar.gz
tar -xvzf flores101_dataset.tar.gz
```

#### Compute spBLEU

Instructions for computing spBLEU for detokenized translations generated by a model 

```bash
flores101_devtest=flores101_dataset/devtest

# Path to generated detokenized translations file
translation_file=/path/to/detok_trans.txt

# Set the target language (for this example, English)
trg_lang=eng

cat $translation_file | sacrebleu -tok spm $flores101_devtest/${trg_lang}.devtest
```

### Example walkthrough of Generation and Evaluation using a pre-trained model in fairseq

Following example walks shows evaluating released `M2M-124 615M` model on an example language pair of `Nyanja -> Swahili` on `FLORES-101` `devtest` which achieves `12.4` spBLEU.

#### Download model, sentencepiece vocab

```bash
fairseq=/path/to/fairseq
cd $fairseq

# Download 615M param model.
wget https://dl.fbaipublicfiles.com/flores101/pretrained_models/flores101_mm100_615M.tar.gz

# Extract 
tar -xvzf flores101_mm100_615M.tar.gz
```

#### Encode using our SentencePiece Model
Note: Install SentencePiece from [here](https://github.com/google/sentencepiece)


```bash
flores101_dataset=/path/to/flores_dataset
fairseq=/path/to/fairseq
cd $fairseq

# Example lang pair translation: Nyanja -> Swahili
# MM100 code for Nyanja and Swahili: ny, sw

SRC_LANG_CODE=nya
TRG_LANG_CODE=swh

SRC_MM100_LANG_CODE=ny
TRG_MM100_LANG_CODE=sw

python scripts/spm_encode.py \
    --model flores101_mm100_615M/sentencepiece.bpe.model \
    --output_format=piece \
    --inputs=$flores101_dataset/devtest/${SRC_LANG_CODE}.devtest \
    --outputs=spm.${SRC_MM100_LANG_CODE}-${TRG_MM100_LANG_CODE}.${SRC_MM100_LANG_CODE}

python scripts/spm_encode.py \
    --model flores101_mm100_615M/sentencepiece.bpe.model \
    --output_format=piece \
    --inputs=$flores101_dataset/devtest/${TRG_LANG_CODE}.devtest \
    --outputs=spm.${SRC_MM100_LANG_CODE}-${TRG_MM100_LANG_CODE}.${TRG_MM100_LANG_CODE}
```

#### Binarization

```bash
fairseq-preprocess \
    --source-lang ${SRC_MM100_LANG_CODE} --target-lang ${TRG_MM100_LANG_CODE} \
    --testpref spm.${SRC_MM100_LANG_CODE}-${TRG_MM100_LANG_CODE} \
    --thresholdsrc 0 --thresholdtgt 0 \
    --destdir data_bin_${SRC_MM100_LANG_CODE}_${TRG_MM100_LANG_CODE} \
    --srcdict flores101_mm100_615M/dict.txt --tgtdict flores101_mm100_615M/dict.txt
```

#### Generation 


```bash
fairseq-generate \
    data_bin_${SRC_MM100_LANG_CODE}_${TRG_MM100_LANG_CODE} \
    --batch-size 1 \
    --path flores101_mm100_615M/model.pt \
    --fixed-dictionary flores101_mm100_615M/dict.txt \
    -s ${SRC_MM100_LANG_CODE} -t ${TRG_MM100_LANG_CODE} \
    --remove-bpe 'sentencepiece' \
    --beam 5 \
    --task translation_multi_simple_epoch \
    --lang-pairs flores101_mm100_615M/language_pairs.txt \
    --decoder-langtok --encoder-langtok src \
    --gen-subset test \
    --fp16 \
    --dataset-impl mmap \
    --distributed-world-size 1 --distributed-no-spawn \
    --results-path generation_${SRC_MM100_LANG_CODE}_${TRG_MM100_LANG_CODE}

# clean fairseq generated file to only create hypotheses file.
cat generation_${SRC_MM100_LANG_CODE}_${TRG_MM100_LANG_CODE}/generate-test.txt  | grep -P '^H-'  | cut -c 3- | sort -n -k 1 | awk -F "\t" '{print $NF}' > generation_${SRC_MM100_LANG_CODE}_${TRG_MM100_LANG_CODE}/sys.txt
```

#### spBLEU Evaluation


```bash
# Get score
sacrebleu flores101_dataset/devtest/${TRG_LANG_CODE}.devtest < generation_${SRC_MM100_LANG_CODE}_${TRG_MM100_LANG_CODE}/sys.txt --tokenize spm
# Expected Outcome:
# BLEU+case.mixed+numrefs.1+smooth.exp+tok.spm+version.1.5.0 = 12.4 34.9/15.8/8.7/4.9 (BP = 1.000 ratio = 1.007 hyp_len = 37247 ref_len = 36999)
```

## List of Languages

Language | FLORES-101 code | MM100 lang code
---|---|---
Akrikaans | afr | af
Amharic | amh | am
Arabic | ara | ar
Armenian | hye | hy
Assamese | asm | as
Asturian | ast | ast
Azerbaijani | azj | az
Belarusian | bel | be
Bengali | ben | bn
Bosnian | bos | bs
Bulgarian | bul | bg
Burmese | mya | my
Catalan | cat | ca
Cebuano | ceb | ceb
Chinese Simpl | zho_simpl | zho
Chinese Trad | zho_trad | zho
Croatian | hrv | hr
Czech | ces | cs
Danish | dan | da
Dutch | nld | nl
English | eng | en
Estonian | est | et
Filipino (Tagalog) | tgl | tl
Finnish | fin | fi
French | fra | fr
Fulah | ful | ff
Galician | glg | gl
Ganda | lug | lg
Georgian | kat | ka
German | deu | de
Greek | ell | el
Gujarati | guj | gu
Hausa | hau | ha
Hebrew | heb | he
Hindi | hin | hi
Hungarian | hun | hu
Icelandic | isl | is
Igbo | ibo | ig
Indonesian | ind | id
Irish | gle | ga
Italian | ita | it
Japanese | jpn | ja
Javanese | jav | jv
Kabuverdianu | kea | kea
Kamba | kam | kam
Kannada | kan | kn
Kazakh | kaz | kk
Khmer | khm | km
Korean | kor | ko
Kyrgyz | kir | ky
Lao | lao | lo
Latvian | lav | lv
Lingala | lin | ln
Lithuanian | lit | lt
Luo | luo | luo
Luxembourgish | ltz | lb
Macedonian | mkd | mk 
Malay | msa | ms
Malayalam | mal | ml
Maltese | mlt | mt
Maori | mri | mi
Marathi | mar | mr
Mongolian | mon | mn
Nepali | npi | ne
Northern Sotho | nso | ns
Norwegian | nob | no
Nyanja | nya | ny
Occitan | oci | oc
Oriya | ory | or
Oromo | orm | om
Pashto | pus | ps
Persian | fas | fa
Polish | pol | pl
Portuguese (Brazil) | por | pt
Punjabi | pan | pa
Romanian | ron | ro
Russian | rus | ru
Serbian | srp | sr
Shona | sna | sn
Sindhi | snd | sd
Slovak | slk | sk
Slovenian | slv | sl
Somali | som | so
Sorani Kurdish | ckb | ku
Spanish (Latin American) | spa | es
Swahili | swh | sw
Swedish | swe | sv
Tajik | tgk | tg
Tamil | tam | ta
Telugu | tel | te
Thai | tha | th
Turkish | tur | tr
Ukrainian | ukr | uk
Umbundu | umb | umb
Urdu | urd | ur
Uzbek | uzb | uz
Vietnamese | vie | vi
Welsh | cym | cy
Wolof | wol | wo
Xhosa | xho | xh
Yoruba | yor | yo
Zulu | zul | zu

## WMT Task
The FLORES-101 dataset is being used for the WMT2021 Large-Scale Multilingual Machine Translation Shared Task. You can learn more about the task [HERE](http://www.statmt.org/wmt21/large-scale-multilingual-translation-task.html). We also provide two pretrained models, downloadable from the WMT task page. 

## Citation

If you use this data in your work, please cite:

```bibtex
@inproceedings{,
  title={The FLORES-101  Evaluation Benchmark for Low-Resource and Multilingual Machine Translation},
  author={Goyal, Naman and Gao, Cynthia and Chaudhary, Vishrav and Chen, Peng-Jen and Wenzek, Guillaume and Ju, Da and Krishnan, Sanjana and Ranzato, Marc'Aurelio and Guzm\'{a}n, Francisco and Fan, Angela},
  year={2021}
}

@inproceedings{,
  title={Two New Evaluation Datasets for Low-Resource Machine Translation: Nepali-English and Sinhala-English},
  author={Guzm\'{a}n, Francisco and Chen, Peng-Jen and Ott, Myle and Pino, Juan and Lample, Guillaume and Koehn, Philipp and Chaudhary, Vishrav and Ranzato, Marc'Aurelio},
  journal={arXiv preprint arXiv:1902.01382},
  year={2019}
}
```

## Changelog
- 2021-06-04: Released FLORES-101

## License
The dataset is licenced under CC-BY-SA, see the LICENSE file for details.
