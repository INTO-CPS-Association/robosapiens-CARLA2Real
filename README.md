# RoboSAPIENS-CARLA2Real: Photorealism Enhancement for CARLA Simulation

**CARLA2Real** is an open-source tool for reducing the simulation-to-reality (sim2real) appearance gap in the [CARLA simulator](https://carla.org/). It enhances simulated camera images using learning-based photorealism enhancement and provides facilities for creating synthetic datasets with synchronized annotations. The original implementation was developed by **Stefanos Pasios**, as part of master's thesis work at Aristotle University of Thessaloniki, and is presented in *CARLA2Real: A Tool for Reducing the Sim2real Appearance Gap in CARLA Simulator* (Pasios and Nikolaidis, 2025).

This **RoboSAPIENS** repository edition organizes the tool documentation for the INTO-CPS ecosystem while retaining the functionality and attribution of the [original CARLA2Real project](https://github.com/stefanos50/CARLA2Real).

---

## Relation to INTO-CPS

This tool is made available in the **INTO-CPS** ecosystem in connection with the European research project **RoboSAPIENS — Robotic Safe Adaptation In Unprecedented Situations**.

CARLA2Real supports the development and evaluation of robotic and autonomous-system perception by:

- reducing the visual-domain gap between simulated and real-world camera observations;
- producing more realistic synthetic images and corresponding ground-truth annotations for perception-model training and evaluation;
- enabling controlled simulation experiments that can help assess the robustness of perception systems under different visual conditions.

The tool focuses on **simulation, photorealism enhancement, and synthetic data generation**. It is not, by itself, a runtime safety monitor or an autonomous adaptation controller.

---

## Method Overview

CARLA2Real integrates the **Enhancing Photorealism Enhancement (EPE)** approach introduced by Intel Labs, which uses simulated rendering information (including G-buffers) to transform CARLA camera frames towards the appearance of real-world datasets. It also includes support for complementary methods:

| Method | Role |
|---|---|
| **EPE** | Photorealism enhancement informed by CARLA rendering buffers. |
| **REGEN** | Faster image-to-image photorealism enhancement. |
| **HyPER-GAN** | Patch-based image-to-image enhancement intended for real-time use. |
| **Hybrid-Sim2Real** | Offline synthetic-data enhancement combining diffusion-generated references with image-to-image translation. |

The repository also supports synchronous dataset extraction, selected autonomous-driving tasks, scenario configuration, and exports of RGB images with semantic/instance segmentation and object-detection annotations. EPE inference can use PyTorch, ONNX Runtime, or TensorRT, according to the selected configuration and installed dependencies. REGEN and HyPER-GAN use PyTorch in the upstream implementation.

---

## Repository Structure

```text
robosapiens-CARLA2Real/
├── code/                          # CARLA2Real implementation and experiments
│   ├── config/                    # EPE, training, testing and CARLA configurations
│   ├── epe/                       # EPE pipeline and dataset-generation tools
│   ├── scenarios/                 # example scenario configurations
│   └── requirements.txt           # implementation dependencies
├── HybridSim2Real/                # hybrid synthetic-image enhancement
├── carla_unreal_engine_5/         # Unreal Engine 5 / CARLA-related material
├── README.md                      # this documentation
├── LICENSE                        # upstream MIT license and acknowledgments
└── CITATION.cff                   # citation metadata
```

Some model checkpoints, datasets, and simulator binaries must be obtained separately and are not bundled with the source repository.

---

## Requirements

- **CARLA:** version **0.9.14 or later** for the `listen_to_gbuffer()` functionality. The upstream documentation recommends **0.9.14** for its tested G-buffer workflow and reports problems with 0.9.15; confirm compatibility before using other versions.
- **Python:** version **3.8** in the upstream installation procedure, preferably through Anaconda/Miniconda.
- **GPU:** a CUDA-capable NVIDIA GPU is required for the supported deep-learning workflows. The upstream project reports approximately **9.5–22.3 GB** of GPU memory depending on model and inference configuration; **10 GB** is a practical minimum for its lower-memory configurations.
- **Platform:** originally developed/tested on Windows 11; functionality on other CARLA-supported platforms can depend on the chosen CUDA, PyTorch, and TensorRT versions.
- **Models:** download the required pretrained checkpoints separately (see [Datasets and Pretrained Models](#datasets-and-pretrained-models)).

---

## Installation

1. Install and start a supported [CARLA simulator](https://carla.org/) instance.
2. Obtain this repository, then create the upstream Conda environment:

   ```bash
   conda create --name epe python=3.8
   conda activate epe
   cd code
   pip install -r requirements.txt
   pip install -e .
   ```

3. Install a **CUDA-compatible PyTorch** build and the remaining GPU dependencies (including FAISS, if needed) according to your hardware and the [original CARLA2Real installation instructions](https://github.com/stefanos50/CARLA2Real#installitationrequirements). The upstream setup also documents installation of `ultralytics` and optional TensorRT/ONNX Runtime support.
4. Download a pretrained model and extract its files under `code/checkpoints/`. Adjust the relevant configuration in `code/config/` to point to the downloaded weights.

For TensorRT, use compatible versions of CUDA, TensorRT, and ONNX Runtime; it is optional for running EPE through PyTorch.

---

## Datasets and Pretrained Models

The upstream project provides links to **synthetic CARLA datasets**, robust label maps, and pretrained photorealism-enhancement models that target **Cityscapes, KITTI, Mapillary Vistas, and nuScenes**.

- [CARLA2Real synthetic dataset (Kaggle)](https://www.kaggle.com/datasets/stefanospasios/carla2real-enhancing-the-photorealism-of-carla)
- [Robust label maps](https://drive.google.com/file/d/1q9Ja6b1mOn_LioDGU-NCYIOKKgDa8wl2/view)
- [Pretrained enhancement models](https://drive.google.com/drive/folders/1WF1RCE-AUWFXdZdWUt3wMbrbBCHrTVCX)
- [Autonomous-driving task models](https://drive.google.com/drive/folders/1ji2lymOjqbNDVS2VExZrSuJtFMaBZ9dI)

The datasets and checkpoints are **external resources**. Check their own access, redistribution, and usage conditions before using or redistributing them.

---

## Running the Experiments

Commands below assume that the Conda environment is active and the working directory is **`code/`**. Launch CARLA first for simulator-based experiments. Review the YAML configuration files for your simulator host, camera settings, model weights, dataset directories, and output paths.

### Real-time enhancement

Run photorealism enhancement on CARLA output:

```bash
python EPEExperiment.py infer config/infer_pfd2cs.yaml --log=info --carla_config config/carla_config.yaml
```

The file `config/carla_config.yaml` controls the simulator integration and selected enhancement method. Set `general/method` to `EPE`, `REGEN`, or `HYPERGAN` as appropriate, and update the corresponding checkpoint-name setting when using REGEN/HyPER-GAN.

### Offline evaluation

Enhance a previously captured CARLA dataset using its images, G-buffers, and labels:

```bash
python EPEExperiment.py test config/test_pfd2cs.yaml --log=info
```

For EPE, **RGB frames alone are not sufficient**: corresponding G-buffers and ground-truth labels are required. See the dataset-path configuration and data-list generation scripts documented upstream.

### Training an EPE model

After preparing the training dataset and updating `config/train_pfd2cs.yaml`, run:

```bash
python EPEExperiment.py train config/train_pfd2cs.yaml --log=info
```

The procedure for creating G-buffer files, label maps, and required text-file lists is described in the [original training documentation](https://github.com/stefanos50/CARLA2Real#training--dataset) and the [Intel Labs EPE project](https://github.com/isl-org/PhotorealismEnhancement).

### Generating a synthetic dataset

The data-generation script is located under `code/epe/dataset_generation/`. For example, from that directory:

```bash
python generate_dataset.py --save_path /path/to/output --ticks_per_frame 10 --perspective 0 --randomize_lights True --town Town10HD
```

Use `python generate_dataset.py --help` to inspect the options. Dataset export in the upstream implementation uses **synchronous mode** to keep images and annotations aligned.

### Hybrid-Sim2Real

For the hybrid method, configure `code/config/carla_config.yaml` (including `export_dataset`, the selected translation method, checkpoint name, and any required Hugging Face access credentials), then run from the repository root:

```bash
cd HybridSim2Real
python enhancer.py
```

See [Hybrid-Sim2Real](https://github.com/stefanos50/Hybrid-Sim2Real) for its method-specific requirements. Never commit access tokens or other credentials to this repository.

---

## Reproducibility

- The upstream implementation and its YAML configuration files provide the reference behavior; results depend on the CARLA build, source/target dataset, checkpoint, GPU, and inference backend.
- The **published timing results are hardware- and precision-dependent**, not guarantees for other systems.
- Keep raw simulator images, G-buffers, annotations, and translated images synchronized when creating evaluation datasets.
- For exact experimental details, refer to the original [CARLA2Real paper](https://doi.org/10.1109/TITS.2025.3597010) and repository documentation.

---

## Known Limitations

- Real-time EPE enhancement requires a CARLA build exposing suitable G-buffers; **0.9.14** is the upstream-tested baseline for this workflow.
- GPU memory and processing requirements can be substantial, especially for high-resolution inference and asynchronous simulator rendering.
- Compatibility and quality can vary across PyTorch/ONNX Runtime/TensorRT backends and numeric precision settings.
- REGEN/HyPER-GAN and Hybrid-Sim2Real have additional checkpoints, dependencies, and configuration requirements.
- External checkpoints, simulator assets, and large datasets are not included in the repository.

---

## License

This project preserves the **MIT License** of the original CARLA2Real implementation, including the copyright notices for **Stefanos Pasios** and the **Intel Embodied AI Lab**. See [LICENSE](LICENSE) for the full license text. External datasets, model checkpoints, CARLA, and other dependencies may be governed by **separate licenses and terms**.

---

## Acknowledgment

This tool is presented as part of the **RoboSAPIENS** project (*Robotic Safe Adaptation In Unprecedented Situations*), funded by the European Union's Horizon Europe research and innovation programme under grant agreement **No. 101133807**. The views and opinions expressed are those of the authors and do not necessarily reflect those of the European Union or the European Commission.

---

## Citation

If you use CARLA2Real in research, please cite the original publication:

```bibtex
@ARTICLE{11122908,
  author={Pasios, Stefanos and Nikolaidis, Nikos},
  journal={IEEE Transactions on Intelligent Transportation Systems},
  title={CARLA2Real: A Tool for Reducing the Sim2real Appearance Gap in CARLA Simulator},
  year={2025},
  volume={26},
  number={11},
  pages={18747-18761},
  doi={10.1109/TITS.2025.3597010}
}
```

Also cite **Enhancing Photorealism Enhancement** (Richter et al., 2023) when using the EPE method. For experiments based on **REGEN**, **HyPER-GAN**, or **Hybrid-Sim2Real**, follow the method-specific citations listed in the [original CARLA2Real README](https://github.com/stefanos50/CARLA2Real#bibtex-citation).
