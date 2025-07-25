# Codec

Pytorch implementation for the training of FAcodec, which was proposed in paper [NaturalSpeech 3: Zero-Shot Speech Synthesis
with Factorized Codec and Diffusion Models](https://arxiv.org/pdf/2403.03100)  

This implementation made some key improvements to the training pipeline, so that the requirements of any form of annotations, including 
transcripts, phoneme alignments, and speaker labels, are eliminated. All you need are simply raw speech files.  
With the new training pipeline, it is possible to train the model on more languages with more diverse timbre distributions.  
We release the code for training and inference, including a pretrained checkpoint on 50k hours speech data with over 1 million speakers.
## Requirements
- Python 3.10

## Installation
```bash
git clone https://github.com/Plachtaa/FAcodec.git
pip install -r requirements.txt
```
If you want to train the model by yourself, install the following packages:
```bash
pip install nemo_toolkit['all']
pip install descript-audio-codec
```

## Model storage
We provide pretrained checkpoints on 50k hours speech data.  

| Model type        | Link                                                                                                                                   |
|-------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| FAcodec           | [![Hugging Face](https://img.shields.io/badge/🤗%20Hugging%20Face-FAcodec-blue)](https://huggingface.co/Plachta/FAcodec)               |
| FAcodec redecoder | [![Hugging Face](https://img.shields.io/badge/🤗%20Hugging%20Face-FAredecoder-blue)](https://huggingface.co/Plachta/FAcodec-redecoder) |

## Demo
Try our model on [![Hugging Face](https://img.shields.io/badge/🤗%20Hugging%20Face-Space-blue)](https://huggingface.co/spaces/Plachta/FAcodecV2)!

## Training
```bash
accelerate launch train.py --config ./configs/config.yaml
```
Before you run the command above, replace the `PseudoDataset` class in `meldataset.py` with your own dataset.
Simply load your own wave files in the same format.  
To train redecoder, the voice conversion model, run:
```bash
accelerate launch train_redecoder.py --config ./configs/config_redecoder.yaml
```
Remember to fill in the checkpoint path of a pretrained FAcodec model in the config file.

## Usage

### Encode & reconstruct
```bash
python reconstruct.py --source <source_wav> --ckpt-path <ckpt_path> --config-path <config_path>
```
If no `--ckpt-path` or `--config-path` is specified, model weights will be automatically downloaded from Hugging Face.  
For China mainland users, add additional environment variable to specify huggingface endpoint:
```bash
HF_ENDPOINT=https://hf-mirror.com python reconstruct_redecoder.py --source <source_wav> --target <target_wav>
```

### Extracting representations
```python
import yaml
from modules.commons import build_model, recursive_munch
from hf_utils import load_custom_model_from_hf
import torch
import torchaudio
import librosa

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
ckpt_path, config_path = load_custom_model_from_hf("Plachta/FAcodec")
model = build_model(yaml.safe_load(open(config_path))['model_params'])
ckpt_params = torch.load(ckpt_path, map_location="cpu")

for key in ckpt_params:
    model[key].load_state_dict(ckpt_params[key])

_ = [model[key].eval() for key in model]
_ = [model[key].to(device) for key in model]

with torch.no_grad():
    source = "path/to/source.wav"
    source_audio = librosa.load(source, sr=24000)[0]
    source_audio = torch.tensor(source_audio).unsqueeze(0).float().to(device)
    z = model.encoder(source_audio[None, ...].to(device).float())
    z, quantized, _, _, timbre, codes = model.quantizer(z, source_audio[None, ...].to(device).float(), return_codes=True)
```
where:  
`timbre` is the timbre representation, one single vector for each utterance.  
`codes[0]` is the prosody representation  
`codes[1]` is the content representation

### Zero-shot voice conversion
```bash
python reconstruct_redecoder.py \
    --source <source_wav> 
    --target <target_wav> 
    --codec-ckpt-path <codec_ckpt_path> 
    --redecoder-ckpt-path <redecoder_ckpt_path> 
    --codec-config-path <codec_config_path> 
    --redecoder-config-path <redecoder_config_path>
```
same as above, if no checkpoint path or config path is specified, model weights will be automatically downloaded from Hugging Face.
