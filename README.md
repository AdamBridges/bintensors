<p align="center">
  <picture>
    <img alt="bintensors" src="https://github.com/GnosisFoundation/bintensors/blob/master/.github/assets/bt_banner.png" style="max-width: 100%;">
  </picture>
</p>

---

<p align="center">
    <a href="https://github.com/GnosisFoundation/bintensors/blob/master/LICENCE.md"><img alt="GitHub" src="https://img.shields.io/badge/licence-MIT Licence-blue"></a>
    <a href="https://crates.io/crates/bintensors"><img alt="Crates.io Version" src="https://img.shields.io/crates/v/bintensors"></a>
    <a href="https://docs.rs/bintensors"><img alt="docs.rs" src="https://img.shields.io/badge/rust-docs.rs-lightgray?logo=rust&logoColor=orange"></a>
    <a href="https://pypi.org/project/bintensors/"><img alt="PyPI" src="https://img.shields.io/pypi/v/bintensors"></a>
    <a href="https://pypi.org/project/bintensors/"><img alt="Python Version" src="https://img.shields.io/pypi/pyversions/bintensors?logo=python"></a>
</p>


Another file format for storing your models and **"tensors"**, in a binary encoded format, designed for speed with zero-copy access.

## Installation

### Cargo

You can add bintensors to your cargo by using `cargo add`:

```bash
cargo add bintensors
```

### Pip

You can install bintensors via the pip manager:

```python
pip install bintensors
```

### From source

For the sources, you need Rust

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# Make sure it's up to date and using stable channel
rustup update
git clone https://github.com/GnosisFoundation/bintensors
cd bintensors/bindings/python
pip install setuptools_rust
# install
pip install -e .
```

## Getting Started

```python
import torch
from bintensors import safe_open
from bintensors.torch import save_file

tensors = {
   "weight1": torch.zeros((1024, 1024)),
   "weight2": torch.zeros((1024, 1024))
}
save_file(tensors, "model.bt")

tensors = {}
with safe_open("model.bt", framework="pt", device="cpu") as f:
   for key in f.keys():
       tensors[key] = f.get_tensor(key)
```

Lets assume we want to handle file in rust

```rust ignore
use std::fs::File;
use memmap2::MmapOptions;
use bintensors::BinTensors;

let filename = "model.bt";
let file = File::open(filename).unwrap();

// Creates a read-only memory map backed by a file.
let buffer = unsafe { MmapOptions::new().map(&file).unwrap() };
// deserialize bytes
let tensors = BinTensors::deserialize(&buffer).unwrap();
let tensor = tensors
        .tensor("weight1");
```

## Overview

This project initially started as an exploration of the `safetensors` file format, primarily to gain a deeper understanding of an ongoing parent project of mine, on distributing models over a subnet. While the format itself is relatively intuitive and well-implemented, it leads to some consideration regarding the use of `serde_json` for storing metadata.

Although the decision by the Hugging Face `safetensors` development team to utilize `serde_json` is understandable, such as for file readability, I questioned the necessity of this approach. Given the complexity of modern models, which can contain thousands of layers, it seems inefficient to store metadata in a human-readable format. In many instances, such metadata might be more appropriately stored in a more compact, optimized format.

**TDLR** why not just use a more otimized serde such as `bincode`.

### Observable Changes

<p align="center">
  <picture>
    <img src="https://github.com/GnosisFoundation/bintensors/blob/master/.github/assets/sf-serde.png" alt="serde safetensors" />
    <sub>Serde figure from safetensors generated by  cargo bench</sub>
  </picture>
  
  <picture>
    <img src="https://github.com/GnosisFoundation/bintensors/blob/master/.github/assets/bt-serde.svg" alt="serde bintensors" />
    <sub>Serde figure from bintensors generated by cargo bench</sub>
  </picture>
</p>

Incorporating the `bincode` library led to a significant performance boost in deserialization, nearly tripling its speed—an improvement that was somewhat expected. Benchmarking code can be found in `bincode/bench/benchmark.rs`, where we conducted two separate tests per repository, comparing the serialization performance of model tests in safesensors and bintensors within the Rust-only implementation. The results, as shown in the figure above, highlight the substantial gains achieved.

To better understand the factors behind this improvement, we analyzed the call stack, comparing the performance characteristics of `serde_json` and `bincode`. To facilitate this, we generated a flame graph to visualize execution paths and identify potential bottlenecks in the `serde_json` deserializer. The findings are illustrated in the figures below.

This experiment was conducted on macOS, and while the results are likely consistent across platforms, I plan to extend the analysis to other operating systems for further validation.

<p align="center">
  <picture>
    <img src="https://github.com/GnosisFoundation/bintensors/blob/master/.github/assets/flamegraph-bt-serde.svg" alt="serde bintensors" />
  </picture>
  <br/>
  <sub>Serde figure from bintensors generated by <a href="https://github.com/flamegraph-rs/flamegraph">flamepgraph</a> & <a href="https://github.com/jonhoo/inferno">inferno</a></sub>
  <br/>
</p>

<p align="center">
  <picture>
    <img src="https://github.com/GnosisFoundation/bintensors/blob/master/.github/assets/flamegraph-sf-serde.svg" alt="serde safetensors" />
    
  </picture>
  <br/>
  <sub>Serde figure from safetensors generated by <a href="https://github.com/flamegraph-rs/flamegraph">flamepgraph</a> & <a href="https://github.com/jonhoo/inferno">inferno</a></sub>
  <br/>
</p>

## Format

<p align="center">
  <picture>
    <img alt="bintensors" src="https://github.com/GnosisFoundation/bintensors/blob/master/.github/assets/bt-format.png" style="max-width: 100%;">
  </picture>
  <br/>

  <sub>
    Visual representation of bintensors (bt) file format
  </sub>
  <br/>
</p>

In this section, I suggest creating a small model and opening a hex dump to better visually decouple it while we go over the high level of the bintensors file format. For a more in-depth understanding of the encoding, I would glance at [specs](https://github.com/GnosisFoundation/bintensors/blob/master/specs/encoding.md).

The file format is divided into three sections:

Header Size:

- 8 bytes: A little-endian, unsigned 64-bit integer that is $2^{63 - 1}$ representing the size of the header, though there is a cap on of 100Mb similar to `safetensors` but that may be changed later in the future.

Header Data

- N bytes: A dynamically serialized table, encoded in a condensed binary format for efficient tensor lookups.
- Contains a map of string-to-string pairs. Note that arbitrary JSON structures are not permitted; all values must be strings.
- The header data deserialization decoding capacity is 100Mb.

Tensor Data

- A sequence of bytes representing the layered tensor data. You can calculate this buffer manually with the formulated equation bellow.

<div style="padding: 0.75em">

$$
B_M = \sum_{t_i \in T}\left( \left[ \prod_{j=1}^{n_i} d_{i,j} \right] \cdot D{\left(x_{i}\right)}\right)
$$

</div>

Let $B_M$ be the total buffer size for a model or model subset $M$. Let $T$ be the set of tensors in the model, where each tensor $t_i$ is an element in this set. Each tensor $t_i$ is characterized by a tuple of 64-bit unsigned integers $(d_{i,1}, d_{i,2}, \dots, d_{i,n_i})$, which represent its shape dimensions The function $D$ is a surjective function, mapping domain of tensor type $x_i$ to the codomain of bytes of tensor dtypes sizes which can be repersented as $\lbrace1, 2, 4, 8\rbrace$.

e.g: let's determine how many bytes are required to store the embedding layer of the GPT-2 model. The model has two large tensors: the **token embedder** (wte) with a shape of $\left(50,257,786\right)$ and the **position embedder** (wpe) with a shape of $\left(1,024,768\right)$. For simplicity, we assume that all weights are stored as ``float32``. This also applies to the ``safetensors`` format.


<div style="padding: 0.75em">

$$ \begin{aligned}
B_{embedding} &= \left(50,257 \times 768\right) \times 4 + \left(1,024 \times 768\right) \times 4 \\
&= (50,257 \times 768) \times 4 + (1,024 \times 768) \times 4 \\
&= 38,597,376 \times 4 + 786,432 \times 4 \\
&= 154,389,504 + 3,145,728 \\
&= 157,535,232 \\
&\therefore B_{encoder} \text{ totals } 157,535,232 \text{ bytes or } 157.54 \text{ MB}
\end{aligned}
$$

</div>

### Notes

- Duplicate keys are disallowed. Not all parsers may respect this.
- Tensor values are not checked against, in particular NaN and +/-Inf could be in the file
- Empty tensors (tensors with 1 dimension being 0) are allowed. They are not storing any data in the databuffer, yet retaining size in the header. They don’t really bring a lot of values but are accepted since they are valid tensors from traditional tensor libraries perspective (torch, tensorflow, numpy, ..).
- The byte buffer needs to be entirely indexed, and cannot contain holes. This prevents the creation of polyglot files.
- Endianness: Little-endian. moment.
- Order: ‘C’ or row-major.
- Checksum over the bytes, giving the file a unique identiy.
  - Allows distributive networks to validate distributed layers checksums.

## Benefits

Since this is a simple fork of safetensors it holds similar propeties that safetensors holds.

- Preformance boost: Bintensors provides a nice preformace boost to the growning ecosystem, of model stroage.
- Prevent DOS attacks: To ensure robust security in our file format, we've implemented anti-DOS protections while maintaining compatibility with the original format's approach. The header buffer is strictly limited to 100MB, preventing resource exhaustion attacks via oversized metadata. Additionally, we enforce strict address boundary validation to guarantee non-overlapping tensor allocations, ensuring memory consumption never exceeds the file's actual size during loading operations. This two-pronged approach effectively mitigates both memory exhaustion vectors and buffer overflow opportunities.
- Faster load: PyTorch seems to be the fastest file to load out in the major ML formats. However, it does seem to have an extra copy on CPU, which we can bypass in this lib by using torch.UntypedStorage.from_file. Currently, CPU loading times are extremely fast with this lib compared to pickle. GPU loading times are as fast or faster than PyTorch equivalent. Loading first on CPU with memmapping with torch, and then moving all tensors to GPU seems to be faster too somehow (similar behavior in torch pickle)
- Lazy loading: in distributed (multi-node or multi-gpu) settings, it’s nice to be able to load only part of the tensors on the various models. For BLOOM using this format enabled to load the model on 8 GPUs from 10mn with regular PyTorch weights down to 45s. This really speeds up feedbacks loops when developing on the model. For instance you don’t have to have separate copies of the weights when changing the distribution strategy (for instance Pipeline Parallelism vs Tensor Parallelism).

Licence: [MIT](https://github.com/GnosisFoundation/bintensors/blob/master/LICENCE.md)
