<div align='center'>
  <img src=https://github.com/user-attachments/assets/9f764ccf-3dce-43c2-b2ae-aa068231dea2 >
</div>

<div align='center'>
  <img src=https://img.shields.io/badge/Language-CUDA/Python-brightgreen.svg >
  <img src=https://img.shields.io/github/watchers/DefTruth/faster-prefill-attention?color=9cc >
  <img src=https://img.shields.io/github/forks/DefTruth/faster-prefill-attention.svg?style=social >
  <img src=https://img.shields.io/github/stars/DefTruth/faster-prefill-attention.svg?style=social >
  <img src=https://img.shields.io/badge/Release-v0.0.1-brightgreen.svg >
  <img src=https://img.shields.io/badge/License-GPLv3.0-turquoise.svg >
 </div>

🤖[WIP] **FFPA**: Yet antother **Faster Flash Prefill Attention** with **O(1) SRAM complexity** & **O(d/4) or O(1) register complexity** for large headdim (D > 256), almost **1.5x~2x** 🎉 faster than SDPA EA with or without MMA Acc F32 on many devices: [📈L20 ~1.7x↑🎉](#L1-bench-l20), [📈 A30 ~1.5x↑🎉](#L1-bench-a30), [📈3080 ~2.5x↑🎉](#L1-bench-3080), [📈4090 ~1.8x↑🎉](#L1-bench-4090). 

<div align='center'>
  <img src='https://github.com/user-attachments/assets/7dc42fa1-a10e-453c-8e2c-befba6f12719' width="407px">
  <img src='https://github.com/user-attachments/assets/c0443e13-94a4-4d29-8f77-e326e62a668e' width="407px">
</div> 

💡NOTE: This project is still in its early dev stages and now provides some kernels and benchmarks for reference. More features will be added in the future. (Welcome to 🌟👆🏻star this repo to support me ~)

## ©️Citations🎉🎉

```BibTeX
@misc{ffpa-attn@2025,
  title={FFPA: Yet another Faster Flash Prefill Attention for large headdim.},
  url={https://github.com/DefTruth/ffpa-attn-mma.git},
  note={Open-source software available at https://github.com/DefTruth/ffpa-attn-mma.git},
  author={DefTruth etc},
  year={2025}
}
```

## 📖 Contents

- [📖 Installation⚙️](#install)
- [📖 Python Testing👇](#python-test)
- [📖 FFPA L1~L3 Design💡](#ffpa-design)
- [📈 FFPA L1: L20 ~1.7x↑🎉](#L1-bench-l20)
- [📈 FFPA L1: A30 ~1.5x↑🎉](#L1-bench-a30)
- [📈 FFPA L1: 3080 ~2.5x↑🎉](#L1-bench-3080)
- [📈 FFPA L1: 4090 ~1.8x↑🎉](#L1-bench-4090)

## 📖 FFPA L1~L3: FlashAttention + QKV Fine-grained Tiling at MMA level💡
<div id="ffpa-design"></div>

We have extended FlashAttention for large headdim (D > 256) by implementing **Fine-grained Tiling** at the **MMA level (GEMM style)** for the Q@K^T and P@V matmul. This approach results in a constant SRAM usage of Br * 16 or Bc * 16 (Br = Bc) for Q, K, and V, leading to an overall SRAM complexity of O(2 * Br * 16) ≈ O(1) and a register complexity of O(d/4) or O(1). Consequently, this method allows us to extend **headdim > 256** and achieve faster performance compared to SDPA with or without MMA Accumulation F32 (**1.5x~2x** 🎉 faster than SDPA EA).

We have named this new attention tiling technique **FFPA: Faster Flash Prefill Attention**. We have designed three `(L1~L3)` levels of FFPA based on SRAM and register complexity considerations. All levels will not introduce any additional VRAM requirements, ensuring that the HBM memory complexity remains same as FlashAttention. 👇

- [x] 📚L1: level 1, O(2xBrx16)≈O(1) SRAM complexity, ≈O(d/4) register complexity.
- [ ] 📚L2: level 2, O(2xBrx16)≈O(1) SRAM complexity, ≈O(1) register complexity + Q@K^T recomputation.
- [ ] 📚L3: level 3, O(2xBrx16)≈O(1) SRAM complexity, ≈O(1) register complexity + scaling O via HBM offloading.

By leveraging this approach, we can achieve better performance for large headdim (D > 256) through a balanced utilization of FlashAttention (which is not designed to support D > 256) and SDPA EA. Approximate SRAM and register complexity analysis for L1~L3 is as follows: (`d`=headdim, `C,Br,Bc`=Constant, `Br=Bc`) 👇

|📚Complexity| 📚FFPA L1 |  📚FFPA L2 |  📚FFPA L3 | 📚FA-2 |
|:---:|:---:|:---:|:---:|:---:|
|SRAM | O(2xBrx16)≈O(1) | O(2xBrx16)≈O(1) | O(2xBrx16)≈O(1) | ≈O(3xBrxd), d↑ |
|Register | ≈O(d/4), d↑ | O((Bc/16)x4+2C)≈O(1)|O((Bc/16)x4+2C)≈O(1)| ≈O(d/2), d↑ |
|HBM| ≈FA2≈O(Nd), O | ≈FA2≈O(Nd), O| ≈FA2≈O(Nd), O | ≈O(Nd), O |
|Extra HBM| ≈FA2≈O(N), m,l | ≈FA2≈O(N), m,l | ≈FA2≈O(N), m,l | ≈O(N), m,l |

**📚👇Core Features🎉🎉**: I have implemented **FFPA L1~L3** using pure MMA PTX instructions, which supports many features such as Split-Q, SMEM Swizzle/Padding, QKV Multi-Stages(1~4), Tile MMAs/Warps, Mixed MMA F32/F16 Acc (Q@K^T MMA Acc F32 + P@V MMA Acc F16), Fully Shared QKV SMEM, Prefetch QKV g2s, **Fully QKV Fine-grained Tiling(GEMM style)**, Collective Store, etc.

|📚Feature |📚Feature |📚Feature |📚Feature|
|:---:|:---:|:---:|:---:|
|✔️Tensor Cores|✔️Loop over N/D |✔️Tile Block(Br, Bc) |✔️**MMA(m16n8k16)**|
|✔️**Split Q**(FA-2)|✔️Pack LDST(128 bits)|✔️SMEM **Swizzle/Pad** |✔️Copy Async |
|✔️Tile MMA/Warp |✔️QKV Multi-Stages(1~4) |✔️Collective Store(**Shfl**)|✔️**Prefetch QKV** g2s |
|✔️**QKV Fine-grained Tiling**|✔️**Shared QKV** SMEM|✔️Mixed MMA Acc|✔️**FFPA L1 Level**|

## 📖 Prerequisites
<div id="prerequisites"></div>

- Python >= 3.10
- PyTorch >= 2.4.0, CUDA >= 12.4
- Recommended: PyTorch 2.5.1, CUDA 12.5
- Docker: nvcr.io/nvidia/pytorch:24.10-py3

## 📖 Installation

<div id="install"></div>

The FFPA implemented in this repo can be install as a python library, namely, `ffpa-attn` library (optional).
```bash
git clone https://github.com/DefTruth/ffpa-attn-mma.git
# clone, then, run bash .dev/install.sh directly or run commands:
python3 setup.py bdist_wheel && cd dist && python3 -m pip install *.whl # pip uninstall ffpa-attn -y
```

## 📖 FFPA L1 (Level 1): Benchmark 🎉🎉

<div id="L1-bench-l20"></div>

L1: level 1, O(2xBrx16)≈O(1) SRAM complexity, O(d/4) register complexity, the same GPU HBM memory complexity as FlashAttention. B=1, H=48, N=8192, **D=320-1024(FA2 not supported 👀)**. (Notes, `*`=MMA Acc F32, `^`=MMA Acc F16, Softmax Acc dtype is always be F32, T=TFLOPS, 👇Benchmark)

- 📚 NVIDIA L20 (`*`=MMA Acc F32, `^`=MMA Acc F16, `T`=TFLOPS, **~1.8x↑🎉**)

|Algorithm|320|384|448|512|576|640|704|768|832|896|960|1024|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|SDPA EA|56T|63T|58T|58T|55T|56T|54T|55T|54T|55T|54T|56T|
|FFPA L1*|102T|102T|103T|104T|103T|95T|95T|95T|95T|96T|95T|94T|
|Speedup|1.82x|1.62x|1.78x|1.79x|1.87x|1.7x|1.76x|1.73x|1.76x|1.75x|1.76x|1.68x|
|FFPA L1^|104T|103T|103T|102T|104T|103T|102T|94T|94T|94T|100T|100T|
|Speedup|1.86x|1.63x|1.78x|1.76x|1.89x|1.84x|1.89x|1.71x|1.74x|1.71x|1.85x|1.79x|

- 📚 NVIDIA L20 (`*`=MMA Acc: QK F32 + PV F16, `^`=MMA Acc F16, `T`=TFLOPS, **~1.9x↑🎉**)

|Algorithm|320|384|448|512|576|640|704|768|832|896|960|1024|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|SDPA EA|56T|64T|58T|58T|55T|56T|54T|55T|54T|55T|54T|56T|
|FFPA L1*|105T|102T|104T|103T|105T|95T|95T|94T|94T|94T|102T|101T|
|Speedup|1.88x|1.59x|1.79x|1.78x|1.91x|1.7x|1.76x|1.71x|1.74x|1.71x|1.89x|1.8x|
|FFPA L1^|104T|103T|103T|102T|103T|103T|102T|94T|94T|94T|100T|100T|
|Speedup|1.86x|1.61x|1.78x|1.76x|1.87x|1.84x|1.89x|1.71x|1.74x|1.71x|1.85x|1.79x|

<div align='center'>
  <img src='https://github.com/user-attachments/assets/7881c7fc-aeb4-4556-92a0-901b5b25ee1b' width="407px">
  <img src='https://github.com/user-attachments/assets/f530900d-0dff-4986-a7e7-47a47ba15556' width="407px">
</div> 

<div id="L1-bench-a30"></div>

- 📚 NVIDIA A30 (`*`=MMA Acc F32, `^`=MMA Acc F16, `T`=TFLOPS, **~1.8x↑🎉**)

|Algorithm|320|384|448|512|576|640|704|768|832|896|960|1024|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|SDPA EA|25T|25T|24T|24T|24T|24T|23T|22T|22T|22T|22T|18T|
|FFPA L1*|45T|44T|44T|43T|43T|38T|37T|37T|37T|36T|33T|32T|
|Speedup|1.8x|1.76x|1.83x|1.79x|1.79x|1.58x|1.61x|1.68x|1.68x|1.64x|1.5x|1.78x|
|FFPA L1^|48T|46T|45T|43T|44T|44T|44T|38T|37T|36T|40T|34T|
|Speedup|1.92x|1.84x|1.88x|1.79x|1.83x|1.83x|1.91x|1.73x|1.68x|1.64x|1.82x|1.89x|

- 📚 NVIDIA A30 (`*`=MMA Acc: QK F32 + PV F16, `^`=MMA Acc F16, `T`=TFLOPS, **~1.9x↑🎉**)

|Algorithm|320|384|448|512|576|640|704|768|832|896|960|1024|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|SDPA EA|25T|25T|24T|24T|24T|24T|23T|22T|22T|22T|22T|18T|
|FFPA L1*|48T|46T|46T|43T|44T|38T|38T|38T|37T|36T|40T|34T|
|Speedup|1.92x|1.84x|1.92x|1.79x|1.83x|1.58x|1.65x|1.73x|1.68x|1.64x|1.82x|1.89x|
|FFPA L1^|48T|46T|45T|43T|44T|44T|44T|38T|37T|36T|39T|34T|
|Speedup|1.92x|1.84x|1.88x|1.79x|1.83x|1.83x|1.91x|1.73x|1.68x|1.64x|1.77x|1.89x|

<div align='center'>
  <img src='https://github.com/user-attachments/assets/7437341e-207d-4e35-b13f-b5834957591f' width="407px">
  <img src='https://github.com/user-attachments/assets/014df0f8-8283-4270-812e-a43bdf10366f' width="407px">
</div> 

<div id="L1-bench-3080"></div>

- 📚 NVIDIA RTX 3080 Laptop (`*`=MMA Acc F32, `^`=MMA Acc F16, `T`=TFLOPS, **~2.5x↑🎉**)

|Algorithm|320|384|448|512|576|640|704|768|832|896|960|1024|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|SDPA EA|13T|16T|12T|16T|15T|15T|14T|14T|14T|14T|14T|14T|
|FFPA L1*|33T|31T|31T|28T|28T|27T|26T|25T|25T|24T|23T|23T|
|Speedup|2.54x|1.94x|2.58x|1.75x|1.87x|1.8x|1.86x|1.79x|1.79x|1.71x|1.64x|1.64x|
|FFPA L1^|41T|40T|39T|38T|37T|36T|36T|34T|33T|32T|30T|30T|
|Speedup|3.15x|2.5x|3.25x|2.38x|2.47x|2.4x|2.57x|2.43x|2.36x|2.29x|2.14x|2.14x|

- 📚 NVIDIA RTX 3080 Laptop (`*`=MMA Acc: QK F32 + PV F16, `^`=MMA Acc F16, `T`=TFLOPS, **~2.8x↑🎉**)

|Algorithm|320|384|448|512|576|640|704|768|832|896|960|1024|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|SDPA EA|13T|16T|12T|16T|15T|15T|15T|15T|14T|14T|14T|14T|
|FFPA L1*|37T|35T|35T|33T|31T|30T|30T|29T|26T|28T|26T|25T|
|Speedup|2.85x|2.19x|2.92x|2.06x|2.07x|2.0x|2.0x|1.93x|1.86x|2.0x|1.86x|1.79x|
|FFPA L1^|41T|41T|40T|39T|38T|37T|36T|35T|32T|31T|30T|31T|
|Speedup|3.15x|2.56x|3.33x|2.44x|2.53x|2.47x|2.4x|2.33x|2.29x|2.21x|2.14x|2.21x|

<div align='center'>
  <img src='https://github.com/user-attachments/assets/7dc42fa1-a10e-453c-8e2c-befba6f12719' width="407px">
  <img src='https://github.com/user-attachments/assets/c0443e13-94a4-4d29-8f77-e326e62a668e' width="407px">
</div> 

<div id="L1-bench-4090"></div>

- 📚 NVIDIA RTX 4090 (`*`=MMA Acc F32, `^`=MMA Acc F16, `T`=TFLOPS, **~1.8x↑🎉**)

|Algorithm|320|384|448|512|576|640|704|768|832|896|960|1024|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|SDPA EA|81T|94T|85T|85T|79T|81T|79T|80T|79T|80T|78T|78T|
|FFPA L1*|149T|150T|150T|150T|150T|140T|140T|140T|139T|139T|137T|134T|
|Speedup|1.84x|1.6x|1.76x|1.76x|1.9x|1.73x|1.77x|1.75x|1.76x|1.74x|1.76x|1.72x|
|FFPA L1^|194T|194T|189T|191T|197T|188T|184T|180T|177T|172T|171T|171T|
|Speedup|2.4x|2.06x|2.22x|2.25x|2.49x|2.32x|2.33x|2.25x|2.24x|2.15x|2.19x|2.19x|

- 📚 NVIDIA RTX 4090 (`*`=MMA Acc: QK F32 + PV F16, `^`=MMA Acc F16, `T`=TFLOPS, **~2.1x↑🎉**)

|Algorithm|320|384|448|512|576|640|704|768|832|896|960|1024|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|SDPA EA|82T|92T|85T|84T|78T|81T|79T|80T|78T|79T|77T|78T|
|FFPA L1*|176T|170T|171T|171T|171T|161T|160T|161T|160T|158T|165T|164T|
|Speedup|2.15x|1.85x|2.01x|2.04x|2.19x|1.99x|2.03x|2.01x|2.05x|2.0x|2.14x|2.1x|
|FFPA L1^|200T|191T|189T|191T|188T|188T|186T|179T|175T|173T|172T|170T|
|Speedup|2.44x|2.08x|2.22x|2.27x|2.41x|2.32x|2.35x|2.24x|2.24x|2.19x|2.23x|2.18x|

<div align='center'>
  <img src='https://github.com/user-attachments/assets/5699465b-03b8-460c-8d9e-7b84bad25d85' width="407px">
  <img src='https://github.com/user-attachments/assets/083a3c6c-1afb-4fc5-9622-34ca22129627' width="407px">
</div> 

## 📖 Python Testing
<div id="python-test"></div>

👇 You can test many custom FFPA kernels via Python and figure out the difference in their performance.
```bash
# You can test on many devices, such as Volta, Ampere, Ada, Hopper, ...
cd tests && python3 test.py --B 1 --H 48 --N 8192 --show-all --D 320
```
- 📚 case: B=1, H=48, N=8192, D=320(`FA2 not supported`)
```bash
python3 test.py --B 1 --H 48 --N 8192 --show-all --D 320
```
- 📚 case: Generate benchmark table on Your own device (Welcome to PR your benchmark table 🎉🎉)
```bash
python3 test.py --gen-bench --show-all
```
- 📚 case: Generate benchmark plots on Your own device (Welcome to PR your benchmark plots 🎉🎉)
```bash
python3 test.py --gen-bench --show-all --plot
```

💡NOTE: Please check all configurable environment variables in [env.py](./env.py).

## ©️License

<div id="License"></div>

GNU General Public License v3.0

## 🎉Contribute

<div id="Contribute"></div>

How to contribute? Wecome to star⭐️ this repo to support me👆🏻 ~

<div align='center'>
<a href="https://star-history.com/#DefTruth/ffpa-attn-mma&Date">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=DefTruth/ffpa-attn-mma&type=Date&theme=dark" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=DefTruth/ffpa-attn-mma&type=Date" />
   <img img width=450 height=300 alt="Star History Chart" src="https://api.star-history.com/svg?repos=DefTruth/ffpa-attn-mma&type=Date" />
 </picture>
</a>
</div>

## 📖 References
<div id="ref"></div>

- [flash-attention](https://github.com/Dao-AILab/flash-attention)
- [CUDA-Learn-Notes](https://github.com/DefTruth/CUDA-Learn-Notes)
- [flashinfer](https://github.com/flashinfer-ai/flashinfer)
