# Enhanced Foundation-Model Interactive Segmentation for Borehole Image Data

**SLB · 2024 · Computer Vision / Semantic Segmentation / Domain Adaptation · Published**
**Publication:** *Enhanced Foundation Model-Based Interactive Segmentation for Borehole
Image Data* — IEEE IGARSS 2025.
**Authorship:** Imane Baho\*, Eya Ghamgui\*, **Abderaouf Boudia**, Franco Marchesoni,
Josselin Kherroubi — *third author*. \*The first two authors are marked as equal
contributors.
**Role:** implementation of the adaptation codebase used for the reported experiments.

📄 [Read the paper](../articles/Paper_IGARSS_InterSEG.pdf)

---

## Problem

Assessing well integrity means detecting corrosion in ultrasonic borehole images. The
imaging tools measure pipe radius and thickness; corrosion appears as metal loss on the
inner or outer wall. The difficulty is that pipes are moulded, so they carry homogeneous
manufacturing patterns and collars that look, to a segmentation model, exactly like the
defects it is supposed to find.

Labelling this data is therefore slow, expensive and requires a domain expert — which is
precisely why there is not enough of it to train a supervised model. Interactive
segmentation breaks the loop: an expert supplies a few clicks and the model does the
rest, so annotation gets cheap enough to bootstrap the dataset.

The obstacle: **SAM performs poorly on this domain**, and WeSAM — the state-of-the-art
weakly-supervised adaptation of SAM — improved it only marginally.

## Context

Two distinct research threads meet here:

- **Interactive segmentation** — guiding a model with sparse human input (clicks, boxes,
  text) rather than dense masks.
- **Source-free domain adaptation** — adapting a pre-trained model to a target domain
  without access to the source data, which is often restricted for privacy, storage or
  confidentiality reasons. WeSAM applies this to SAM with a self-training architecture:
  frozen **anchor**, **student** and **teacher** encoders, weak and strong augmentations,
  anchor-loss regularisation and a contrastive loss.

## The paper's contributions

**1 — Clicking procedure.** WeSAM derives its training prompts by randomly sampling 5
positive and 5 negative points inside and outside the ground-truth mask. That is not how
a human annotates. The paper adopts the iterative click simulation of RITM: the first
click seeds the mask, and each subsequent click is placed at the centroid of the largest
error region between the current prediction and the ground truth. Ten clicks are selected
this way, so training sees the corrections a real annotator would make.

**2 — Prompt encoding.** SegNext's *dense* prompt encoding is brought into the WeSAM
architecture. Visual prompts become a 3-channel dense map — positive clicks in the first
channel, negative clicks in the second, the previous mask in the third — fused with the
image feature maps and passed through self- and cross-attention before the mask decoder.
Encoding the previous mask lets each interaction refine the last one instead of starting
over.

**3 — Customised contrast augmentation.** A model trained on this data learns to rely on
high contrast between groove and background. A custom contrast shift was added to the
strong-augmentation set specifically to create overlap between the two, removing that
shortcut.

## Dataset

**THBK** (Azimuthal Variation of Pipe Thickness) — 180 grayscale ultrasonic images with
binary masks, targeting axial long defects ("grooves"). Acquisition corrupts some
samples, so cleaning and pre-processing precede training. Augmentations are split into
weak (vertical/horizontal flips) and strong (posterisation, sharpening, random brightness
and contrast, plus the customised contrast shift).

## Results

IoU (%) as a function of the number of user clicks, as reported in the paper:

| Method | 1 click | 10 clicks | 20 clicks |
|---|---|---|---|
| SAM | 5.02 | 43.81 | 49.68 |
| WeSAM | 2.64 | 51.57 | 52.15 |
| Ours 1 *(clicking procedure)* | 2.51 | 54.67 | 56.79 |
| SegNext | 8.55 | 52.32 | 73.83 |
| **Ours 2** *(both contributions)* | **14.14** | **65.34** | **83.95** |

The abstract reports a gain of **over 13 IoU points**. Ablation confirms the customised
contrast shift contributes a consistent improvement at every click budget (e.g. 65.34 vs
60.03 at 10 clicks).

The first contribution alone improves on WeSAM; the combined approach surpasses SegNext
at every click count, and qualitatively reduces false positives — which is what matters
when the background is a manufacturing pattern that mimics the defect.

## My technical contribution

Stated separately from bibliographic authorship, which is recorded above.

I implemented the adaptation codebase used to produce the reported experiments:

- The **iterative click simulator** — a `Clicker` that maintains click state, computes the
  largest error region between prediction and ground truth, and places the next click at
  its centroid (the paper's first contribution).
- The **two adaptation procedures** — the error-driven iterative clicking procedure and
  the random positive/negative point baseline it is compared against.
- The **self-training loop**: anchor / student / teacher forward passes over weak and
  strong augmentations, with focal, Dice, IoU, anchor-regularisation and contrastive
  losses, and **LoRA** fine-tuning applied to the SAM backbone.
- **Dataset adapters** for the evaluation corpora, and the **inference, evaluation and
  plotting tooling** used to produce the click-budget curves and qualitative comparisons.

The work builds on the publicly released WeSAM codebase (Zhang et al., CVPR 2024); the
contributions above are implemented on top of it.

## Technologies

Python · PyTorch · PyTorch Lightning · Segment Anything Model (SAM) · LoRA (low-rank
adaptation) · SegNext-style dense prompt encoding · RITM-style click simulation ·
segmentation-models-pytorch · Albumentations · OpenCV · pycocotools · TensorBoard

## What I learned

- **How you simulate the human is part of the method.** Replacing random point sampling
  with error-driven clicks changed results more than any loss-weight tuning.
- **Prompt representation is architecture.** Moving from sparse to dense prompt encoding,
  and feeding the previous mask back in, is where the large gain came from.
- **Augmentation can remove a shortcut, not just add variance.** The contrast shift was
  designed against a specific bias the model was exploiting.

## Links

- 📄 [Paper (PDF)](../articles/Paper_IGARSS_InterSEG.pdf) — IEEE IGARSS 2025
- Base method: Zhang, Su, Xu & Jia, *Improving the Generalization of Segmentation
  Foundation Model under Distribution Shift via Weakly Supervised Adaptation*, CVPR 2024.
- Implementation code is company-internal and is not published.
