---
layout: post
comments: true
published: true
title: Username Grapheme to Phoneme Transformer
description: Better username pronunciation for Text-to-Speech
---

I've been running text-to-speech on my Discord bot for almost 8 years now. I mainly use it to announce when users join/leave a channel, similar to how Ventrillo works. I started out using gTTS, but later moved to using free ElevenLabs credits whenever available (still defaulting back to good old gTTS when the credits run out). While the TTS is pretty good, it still has trouble pronouncing some usernames. I thought this might be a great time to try training a Seq2Seq transformer model, taking usernames as input and outputting *some kind* of pronunciation data.

I quickly found the [CMU Pronouncing Dictionary](http://www.speech.cs.cmu.edu/cgi-bin/cmudict), a well-established mapping of words to a special phonetic alphabet, called an ARPAbet. Additionally, Python's nltk module also allows easy access to the dataset. I also found a data set of [10 million usernames](https://github.com/danielmiessler/SecLists/tree/master/Usernames), originally for security research, but I thought it might be useful to specifically train a model to pronounce usernames, instead of general words.

I started with a Google Colab instance, running on a Tesla T4 GPU, with high RAM. We install the required libraries, including the extra torch libraries to allow for CUDA support.

```python
# Install PyTorch
%pip install torch torchvision torchaudio

# Install other dependencies
%pip install numpy pandas nltk elevenlabs requests
```

Next, we download the CMU Pronouncing Dictionary

```python
import nltk

nltk.download('cmudict')
```

This might take a few minutes. Once it's done, we can download the usernames dataset. For reasons I'll explain later, I'm only going to download the first 750,000 usernames.

```python
url = "https://raw.githubusercontent.com/danielmiessler/SecLists/master/Usernames/xato-net-10-million-usernames.txt"

response = requests.get(url)
response.raise_for_status()  # Raise an exception for bad status codes

usernames = response.text.splitlines()[:750_000]
print(f"Downloaded {len(usernames)} usernames.")

```

We need to preprocess the usernames, removing any special characters, numbers, converting common spacers to actual spaces, and setting them to lowercase.

```python
import re
def normalize_username(username):
    # Convert to lowercase
    username = username.lower()
    # Replace numbers with words
    num_to_word = {
        '0': ' zero ', '1': ' one ', '2': ' two ', '3': ' three ',
        '4': ' four ', '5': ' five ', '6': ' six ', '7': ' seven ',
        '8': ' eight ', '9': ' nine '
    }
    for num, word in num_to_word.items():
        username = username.replace(num, word)
    # Replace special characters with spaces
    username = re.sub(r'[\W_]+', ' ', username)
    # Remove extra spaces
    username = re.sub(r'\s+', ' ', username).strip()
    return username
```

We need to make sure we only include usernames that have a pronunciation in the CMU Pronouncing Dictionary. We can do this by checking if the username is in the dictionary. Additionally, multiple pronunciations are possible for a single word, so we'll only use the first one.

```python
def get_phonemes(word):
    phonemes_list = cmu_dict.get(word)
    if phonemes_list:
        return phonemes_list[0]  # Use the first pronunciation
    else:
        return None  # Only show usernames that have correct phonemes
```

Finally, we can process each username, and get the corresponding phonemes.

```python
def username_to_phonemes(username):
    normalized = normalize_username(username)
    words = normalized.split()
    phonemes = []
    for word in words:
        phoneme = get_phonemes(word)
        if phoneme:
            phonemes.extend(phoneme)
    return phonemes
```

Now we can finally preprocess the usernames and get the phonemes.

```python
input_sequences = []
target_sequences = []

for username in usernames:
    input_seq = list(normalize_username(username))
    target_seq = username_to_phonemes(username)
    if target_seq:
      input_sequences.append(input_seq)
      target_sequences.append(target_seq)
```

We need to pad the sequences to the same length, and get the characters of both the input and output sequences.

```python
# Character Vocabulary
char_counter = Counter([char for seq in input_sequences for char in seq])
char_list = ['<pad>'] + sorted(char_counter.keys())
char_vocab = {char: idx for idx, char in enumerate(char_list)}

# Phoneme Vocabulary
phoneme_counter = Counter([phoneme for seq in target_sequences for phoneme in seq])
phoneme_list = ['<pad>', '<sos>', '<eos>'] + sorted(phoneme_counter.keys())
phoneme_vocab = {phoneme: idx for idx, phoneme in enumerate(phoneme_list)}

def encode_sequence(seq, vocab, max_len, add_special_tokens=False):
    encoded = [vocab.get(token, vocab['<pad>']) for token in seq]
    if add_special_tokens:
        encoded = [vocab['<sos>']] + encoded + [vocab['<eos>']]
    # Trim or pad the sequence to max_len
    encoded = encoded[:max_len] + [vocab['<pad>']] * max(0, max_len - len(encoded))
    return encoded


max_input_len = max(len(seq) for seq in input_sequences)
max_target_len = max(len(seq) for seq in target_sequences) + 2  # For <sos> and <eos>

encoded_inputs = [encode_sequence(seq, char_vocab, max_input_len) for seq in input_sequences]
encoded_targets = [encode_sequence(seq, phoneme_vocab, max_target_len, True) for seq in target_sequences]
```

We can now create a simple PyTorch dataset and dataloader to handle the data.

```python
class UsernameDataset(Dataset):
    def __init__(self, inputs, targets):
        self.inputs = torch.tensor(inputs, dtype=torch.long)
        self.targets = torch.tensor(targets, dtype=torch.long)

    def __len__(self):
        return len(self.inputs)

    def __getitem__(self, idx):
        return self.inputs[idx], self.targets[idx]

dataset = UsernameDataset(encoded_inputs, encoded_targets)
data_loader = DataLoader(dataset, batch_size=64, shuffle=True)
```

We'll create the encoder, attention, and decoder blocks.

```python
import torch
import torch.nn as nn

class Encoder(nn.Module):
    def __init__(self, input_dim, emb_dim, hid_dim):
        super().__init__()
        self.embedding = nn.Embedding(input_dim, emb_dim, padding_idx=char_vocab['<pad>'])
        self.gru = nn.GRU(emb_dim, hid_dim, batch_first=True)

    def forward(self, src):
        embedded = self.embedding(src)
        outputs, hidden = self.gru(embedded)
        return outputs, hidden

class Attention(nn.Module):
    def __init__(self, hid_dim):
        super().__init__()
        self.attn = nn.Linear(hid_dim * 2, hid_dim)
        self.v = nn.Linear(hid_dim, 1, bias=False)

    def forward(self, hidden, encoder_outputs):
        src_len = encoder_outputs.shape[1]
        hidden = hidden.repeat(1, src_len, 1)
        energy = torch.tanh(self.attn(torch.cat((hidden, encoder_outputs), dim=2)))
        attention = self.v(energy).squeeze(2)
        return torch.softmax(attention, dim=1)

class Decoder(nn.Module):
    def __init__(self, output_dim, emb_dim, hid_dim, attention):
        super().__init__()
        self.output_dim = output_dim
        self.attention = attention
        self.embedding = nn.Embedding(output_dim, emb_dim, padding_idx=phoneme_vocab['<pad>'])
        self.gru = nn.GRU(emb_dim + hid_dim, hid_dim, batch_first=True)
        self.fc_out = nn.Linear(hid_dim * 2, output_dim)

    def forward(self, input, hidden, encoder_outputs):
        input = input.unsqueeze(1)
        embedded = self.embedding(input)
        a = self.attention(hidden.permute(1, 0, 2), encoder_outputs)
        a = a.unsqueeze(1)
        weighted = torch.bmm(a, encoder_outputs)
        rnn_input = torch.cat((embedded, weighted), dim=2)
        output, hidden = self.gru(rnn_input, hidden)
        output = torch.cat((output.squeeze(1), weighted.squeeze(1)), dim=1)
        prediction = self.fc_out(output)
        return prediction, hidden
```

Finally, we can create the Seq2Seq model.

```python
import numpy as np

class Seq2Seq(nn.Module):
    def __init__(self, encoder, decoder, device):
        super().__init__()
        self.encoder = encoder
        self.decoder = decoder
        self.device = device

    def forward(self, src, trg, teacher_forcing_ratio=0.5):
        batch_size = src.shape[0]
        trg_len = trg.shape[1]
        trg_vocab_size = self.decoder.output_dim

        outputs = torch.zeros(batch_size, trg_len, trg_vocab_size).to(self.device)
        encoder_outputs, hidden = self.encoder(src)
        input = trg[:, 0]

        for t in range(1, trg_len):
            output, hidden = self.decoder(input, hidden, encoder_outputs)
            outputs[:, t] = output
            top1 = output.argmax(1)
            teacher_force = np.random.random() < teacher_forcing_ratio
            input = trg[:, t] if teacher_force else top1
        return outputs
```

Since we're working out of colab, we want to periodically save the model. We'll use Google Drive to persist the data, using some regex to get the latest checkpoint file and increment the version number.

```python
from google.colab import drive
import os

drive.mount('/content/drive')

def get_latest_checkpoint(directory):
    # Get a list of all files in the directory
    files = os.listdir(directory)

    # Filter the list to only include g2p{n}.pth files
    checkpoint_files = [f for f in files if re.match(r'g2p\d+\.pth', f)]

    # Extract the numbers from the filenames
    checkpoint_numbers = [int(re.search(r'g2p(\d+)\.pth', f).group(1)) for f in checkpoint_files]

    # Sort the files by their numbers
    sorted_files = sorted(zip(checkpoint_numbers, checkpoint_files))

    # Get the latest file (last element in the sorted list)
    if sorted_files:
        latest_file = sorted_files[-1][1]
        latest_checkpoint_path = os.path.join(directory, latest_file)
        return latest_checkpoint_path
    else:
        return None

def get_next_version(directory):
    files = os.listdir(directory)

    # Filter the list to only include g2p{n}.pth files
    checkpoint_files = [f for f in files if re.match(r'g2p\d+\.pth', f)]

    # Extract the numbers from the filenames
    checkpoint_numbers = [int(re.search(r'g2p(\d+)\.pth', f).group(1)) for f in checkpoint_files]

    # Sort the files by their numbers
    sorted_files = sorted(zip(checkpoint_numbers, checkpoint_files))
    if sorted_files:
        latest_version = sorted_files[-1][0]
        print(f"Latest version: {sorted_files[-1]}")
        return latest_version + 1
    else:
        return 1  # Start with version 1 if no checkpoints exist

def save_checkpoint(model, directory, version):
    filename = f"g2p{version}.pth"
    filepath = os.path.join(directory, filename)
    torch.save(model.state_dict(), filepath)
    print(f"Model saved to {filepath}")

# Get the latest checkpoint file path
directory = '/content/drive/MyDrive/AI/username_g2p/'
latest_checkpoint_file = get_latest_checkpoint(directory)

if latest_checkpoint_file:
    print(f"Latest checkpoint file: {latest_checkpoint_file}")
else:
    print("No checkpoint files found.")
```

We define our model and training parameters and load the latest checkpoint if it exists.

```python
INPUT_DIM = len(char_vocab)
OUTPUT_DIM = len(phoneme_vocab)
ENC_EMB_DIM = 64
DEC_EMB_DIM = 64
HID_DIM = 128

attn = Attention(HID_DIM)
enc = Encoder(INPUT_DIM, ENC_EMB_DIM, HID_DIM)
dec = Decoder(OUTPUT_DIM, DEC_EMB_DIM, HID_DIM, attn)

device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = Seq2Seq(enc, dec, device).to(device)
optimizer = torch.optim.Adam(model.parameters())
criterion = nn.CrossEntropyLoss(ignore_index=phoneme_vocab['<pad>'])

# Path to your checkpoint file
checkpoint_file = latest_checkpoint_file if latest_checkpoint_file else 'g2p1.pth'

# Check if the checkpoint file exists
if os.path.exists(checkpoint_file):
    # Load the checkpoint
    print(f"Loading checkpoint from {checkpoint_file}")
    model.load_state_dict(torch.load(checkpoint_file))
else:
    print(f"Checkpoint file not found. Using default initialization.")
```

Our training loop is pretty standard, with the addition of the tqdm progress bar.

```python

def train(model, loader, optimizer, criterion, clip):
    model.train()
    epoch_loss = 0

    for src, trg in tqdm(loader, desc="Training Batches"):
        src, trg = src.to(device), trg.to(device)
        optimizer.zero_grad()
        output = model(src, trg)
        output_dim = output.shape[-1]
        output = output[:, 1:].reshape(-1, output_dim)
        trg = trg[:, 1:].reshape(-1)
        loss = criterion(output, trg)
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), clip)
        optimizer.step()
        epoch_loss += loss.item()

    return epoch_loss / len(loader)
```

Finally, we can train the model. I like to run for ~7 epochs, which goes for ~15 minutes per epoch on a Tesla T4. This way I don't have to babysit the training process, but it won't time out.

```python
N_EPOCHS = 7
CLIP = 1

for epoch in range(N_EPOCHS):
    loss = train(model, data_loader, optimizer, criterion, CLIP)
    print(f'Epoch: {epoch+1}, Loss: {loss:.4f}')

# Get the next version number
next_version = get_next_version(directory)

# Save the model with the new version number
save_checkpoint(model, directory, next_version)
```

We can now use the model to predict the phonemes of a username by passing it through the model.

```python
def predict(model, username):
    model.eval()
    with torch.no_grad():
        normalized = normalize_username(username)
        input_seq = encode_sequence(list(normalized), char_vocab, max_input_len)
        src = torch.tensor([input_seq], dtype=torch.long).to(device)
        encoder_outputs, hidden = model.encoder(src)
        input_token = torch.tensor([phoneme_vocab['<sos>']], dtype=torch.long).to(device)
        outputs = []

        for _ in range(max_target_len):
            output, hidden = model.decoder(input_token, hidden, encoder_outputs)
            top1 = output.argmax(1)
            if top1.item() == phoneme_vocab['<eos>']:
                break
            outputs.append(top1.item())
            input_token = top1

        idx_to_phoneme = {idx: phoneme for phoneme, idx in phoneme_vocab.items()}
        predicted_phonemes = [idx_to_phoneme[idx] for idx in outputs]
        return ' '.join(predicted_phonemes)
```

Now, when our epoch set is done, we can test the model on some usernames.

```python
test_username = 'barnabassacket'
pronunciation = predict(model, test_username)
print(f'Username: {test_username}')
print(f'Pronunciation: {pronunciation}')
```

 ```Pronunciation: B AA1 R N AH0 B AE2 S K AH0 T```

At this point, we could also convert to IPA using [this lookup](https://github.com/margonaut/CMU-to-IPA-Converter/blob/master/cmu_ipa_mapping.rb) table.

```ruby
CMU_IPA_MAPPING = {
  B: "b",
  CH: "ʧ",
  D: "d",
...
  OW2: "oʊ",
  OY0: "ɔɪ",
  OY1: "ɔɪ",
  OY2: "ɔɪ"
}
```

The file's in Ruby, but here's [the Python](https://gist.github.com/samclane/6c6c1ad695bf623988e86164bee2b09b). Now you can just write

```python
pronunciation = predict(model, test_username)
ipa_sequence = ''.join([CMU_IPA_MAPPING.get(phoneme, phoneme) for phoneme in pronunciation.split()])
print(f'Username: {test_username}')
```

In order to feed the output to Eleven Labs, we use their "Speech Synthesis Markup Language" (SSML) to control the pronunciation.

```html
<phoneme alphabet="ipa" ph="ˈæktʃuəli">actually</phoneme>
```

In python, we can configure a quick string template like so:

```python
ssml_template = """<phoneme alphabet="{alphabet}" ph="{phonetics}">{text}</phoneme>"""

class Alphabets:
  IPA = "ipa"
  CMU = "cmu-arpabet"

print(ssml_template.format(alphabet=Alphabets.IPA, phonetics="ˈæktʃuəli", text="actually"))
```

To pass the SSML to Eleven Labs, we can use the `elevenlabs` library, along with an API key we can set up and pull from the Google colab environment.

```python
from google.colab import userdata
eleven_labs_key = userdata.get('ELEVENLABS')
from elevenlabs import save
from elevenlabs.client import ElevenLabs
from IPython.display import Audio, display

sound_file = 'test.mp3'

def build_eleven_labs_query(username: str):
  client = ElevenLabs(
    api_key=eleven_labs_key,
  )

  audio = client.generate(
    text=ssml_template.format(
        alphabet=Alphabets.CMU,
        phonetics=predict(model, username),
        text=username
    ),
    voice="Rachel",
    model="eleven_flash_v2"
  )
  save(audio, sound_file)

build_eleven_labs_query(test_username)


display(Audio(sound_file, autoplay=True))
```

If running in a notebook, you'll hear the sound file play. If you're running this in a script, you can just play the file `test.mp3` locally.

I ran this for about 100 total epochs, over a few days, and the results were pretty good. I was able to get a lot of the usernames to pronounce correctly, and the ones that didn't were usually due to them being overly long (for example `supercalafragalisticexpialadocous` outputs `S AH0 P ER0 K AE1 L AH0 S AH0 S AH0 S`), but it worked surprisingly well for most usernames in my Discord server.

There are many useful papers on this topic, but I found [this one](https://arxiv.org/pdf/2104.04091) useful in understanding the current state of the art. Existing models such as `SpeechBrain` are much more complex and powerful than mine, and can be found at [this HuggingFace page](https://huggingface.co/speechbrain/soundchoice-g2p).

The HuggingFace Model can be found [here](https://huggingface.co/samclane/usernameg2p).

The processed username dataset can be found here [here](https://huggingface.co/datasets/samclane/username_pronunciation).
