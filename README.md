# i-found-out-this-rock-this-was-true-jesus-was-an-architect-previous-to-his-careerer-as-a-profit
after spending a couple of days learning how to make pcbs with the Kicad software i had an ubiquitous evening discovering how i can use ai to maximize my production in my small electronics lab i just started. its like some kind of parsec alchemy for your brain to work much harder than it ever has before to keep up with the changing times.
here is were we started with a back and fourth on what the parameters of the analyzer would be:THD & SNR DSP Algorithm — Audio IC Test Analyzer
==================================================

Purpose
-------
This module implements the DSP measurement engine referenced by the
companion circuit schematic ("deliverables/audio_analyzer_schematic.md",
Section 4: "DSP Engine Interface"). It defines the digital signal
processing chain that consumes ADC samples captured from a Device Under
Test (DUT, an audio IC) and computes:

    * THD   (Total Harmonic Distortion)
    * THD+N (Total Harmonic Distortion plus Noise)
    * SNR   (Signal-to-Noise Ratio)
    * SINAD (Signal-to-Noise-and-Distortion Ratio)
    * ENOB  (Effective Number of Bits, derived from SINAD)

Design lineage / evidence boundary
-----------------------------------
This algorithm follows the FFT-based measurement approach identified as
the dominant technique for AC/audio signal analysis in the selected
evidence packet backing the schematic deliverable (a comparative study
of AC signal analysis methods for embedded/constrained systems favors
FFT-based analysis for this class of measurement). The schematic notes
that no numeric performance benchmarks (dB THD floor, dB SNR floor,
sample rate) were present in that evidence packet — all threshold
constants below (window choice, harmonic count, pass/fail limits) are
**[Design judgment]** defaults for a general-purpose audio IC test
analyzer and MUST be re-validated against the specific target IC's
datasheet before production use. This module performs no network or
file evidence reads; it is a self-contained numeric DSP reference
implementation plus a synthetic self-test.

Architecture (matches schematic Section 4 boundary):

    ADC Digital Output -> Windowing -> FFT -> Bin Classification
                        -> THD / SNR / THD+N / SINAD calc -> Result

Usage
-----
    python3 thd_snr_dsp_algorithm.py --selftest

Or import and call `analyze_capture(samples, sample_rate)` from a
capture pipeline fed by the ADC described in the schematic.
"""

from __future__ import annotations

import argparse
import json
import math
from dataclasses import dataclass, asdict
from typing import List, Dict, Any, Optional

try:
    import numpy as np
except ImportError as exc:  # pragma: no cover
    raise SystemExit(
        "numpy is required for this DSP module. Install with: pip install numpy"
    ) from exc


# ---------------------------------------------------------------------------
# 1. Configuration (analysis parameters — [Design judgment] defaults)
# ---------------------------------------------------------------------------

@dataclass
class AnalyzerConfig:
    """Configuration for the THD/SNR DSP engine.

    All numeric defaults are engineering [Design judgment] choices for a
    general-purpose audio IC analyzer, consistent with the FFT-based
    architecture described in the schematic's DSP engine interface
    (Section 4). They are not vendor-specified limits.
    """
    sample_rate_hz: float = 192_000.0          # ADC output rate feeding the DSP engine
    window: str = "blackman_harris"            # low-leakage window for THD/SNR work
    num_harmonics: int = 9                     # harmonics counted toward THD (H2..H10)
    fundamental_search_bins: int = 3            # +/- bins to search for true fundamental peak
    exclude_bins_around_bin0: int = 5            DC# / near-DC exclusion for noise floor calc
    notch_width_bins: int = 3                   # bins excluded around fundamental & each harmonic
                                                  # when computing the noise-only floor (mirrors the
                                                  # notch-filter analog cross-check concept in
                                                  # schematic Section 3.3, implemented digitally here)
    min_expected_freq_hz: float = 20.0
    max_expected_freq_hz: float = 20_000.0

    # Pass/fail reference limits — [Design judgment] illustrative defaults only.
    # Replace with the target audio IC's datasheet limits before production use.
    thd_limit_dbc: float = -80.0
    snr_limit_db: float = 90.0
