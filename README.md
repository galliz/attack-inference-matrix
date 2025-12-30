# ATT&CK Inference Matrix

[![MITRE ATT&CK® v15](https://img.shields.io/badge/MITRE%20ATT%26CK®-v15-red)](https://attack.mitre.org/versions/v15/)

A unified web application that integrates the [Technique Inference Engine (TIE)](https://center-for-threat-informed-defense.github.io/technique-inference-engine/) with the [MITRE ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/), allowing instant visualization of inferred techniques directly on the ATT&CK matrix.

## Features

- **Interactive technique selection**: Select observed techniques directly on the ATT&CK Navigator matrix
- **Real-time inference**: Visualize predicted techniques on the same matrix without exporting files

## How it works

1. Select observed ATT&CK techniques on the matrix
2. View predicted techniques highlighted directly on the matrix

## About the Technique Inference Engine

TIE enables cyber defenders to forecast adversary next steps by predicting associated MITRE ATT&CK techniques from previously observed techniques. The engine is trained on 43,899 technique observations across 6,236 CTI Reports, achieving 96% coverage of ATT&CK Enterprise v15.0.

## Credits

This project builds upon:
- [Technique Inference Engine](https://github.com/center-for-threat-informed-defense/technique-inference-engine) by MITRE Center for Threat-Informed Defense
- [ATT&CK Navigator](https://github.com/mitre-attack/attack-navigator) by MITRE

## Notice

Copyright 2024 MITRE. Approved for public release. Document number CT0124.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this
file except in compliance with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under
the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied. See the License for the specific language governing
permissions and limitations under the License.
