# Bachelor and Master Thesis backup

## Description

This folder contains the thesis documents of Bachelor and Master Thesis of students that have contributed to the development of the project. 

Some of the thesis are in Spanish, others in English. For some of them there are also videos that explain the work done. The original source codes are also referenced despite they might not be the latest version of the code. 

## Where the device work lives now

The device work used to live in a single repository, `hemoglobulab_device`, with one
branch per concern. That repository is now **archived** (read-only) and its contents
have been split into dedicated repositories:

| What | Was | Is now |
| --- | --- | --- |
| Device firmware (current) | did not exist yet | [bloodtwin_device_fw](https://github.com/greenlsi/bloodtwin_device_fw), Zephyr / nRF Connect SDK |
| Device firmware v1 (nRF5 SDK) | `hemoglobulab_device`, branch `master` (`device-fw/`) | [bloodtwin_device_fw](https://github.com/greenlsi/bloodtwin_device_fw), branch `legacy/nrf5-sdk-firmware` |
| Signal processing and ML models | `hemoglobulab_device`, branch `devel_modeling` | [bloodtwin_device_models](https://github.com/greenlsi/bloodtwin_device_models) |
| Board design, Gerbers and BOM | never in git, only in Drive | [bloodtwin_device_hw](https://github.com/greenlsi/bloodtwin_device_hw) |
| Android app | `hemoglobulab_app` (its own repository) | **archived**, superseded by [bloodtwin_webapp_frontend](https://github.com/greenlsi/bloodtwin_webapp_frontend) and [bloodtwin_webapp_backend](https://github.com/greenlsi/bloodtwin_webapp_backend) |
| Android BLE reference library | `hemoglobulab_device`, branch `devel` (`android-app/`) | **archived only**, it is only a vendored copy of the Nordic example |

Nothing was deleted. The archived repository keeps its full history, and each branch is
also reachable through a tag: `archive/device-firmware`, `archive/data-modeling` and
`archive/android-ble-reference`.

The v1 firmware was moved with its own history preserved: the 20 commits that only touch
`device-fw/` were extracted and pushed to `bloodtwin_device_fw` as the branch
`legacy/nrf5-sdk-firmware`, tagged `archive/andres-moreno-nrf5-sdk`. It targets the
nRF52840 DK with the old nRF5 SDK and Eclipse/GCC toolchain, and is superseded by the
current Zephyr firmware on `main`.

The Android app is intentionally **not** carried over into the new structure. It is
superseded by the web application and is kept purely as a historical record, in its own
archived [hemoglobulab_app](https://github.com/greenlsi/hemoglobulab_app) repository,
tagged `archive/android-app-tfg-amoreno`.

Note on the modelling code: the 2023 exploration in `devel_modeling` was left unfinished.
The version of the code that actually produced the published results is the one now in
`bloodtwin_device_models`, under `src/`, with the earlier exploration preserved in
`legacy/`.

## Thesis

- Nacho Uranga: [Master Thesis](https://youtu.be/VVGAXvXanxs) on the DEVS simulator. The original Java implementation is preserved in the archived [hemoglobulab_MSO/hemoglobulsim](https://github.com/greenlsi/hemoglobulab_MSO/tree/master/hemoglobulsim) repository. The later Python prototype is preserved in the archived [crhm_sim](https://github.com/greenlsi/crhm_sim) repository, tagged `archive/crhm-sim-2023`. Early `bdslib` work is retained as historical branches of [bloodtwin_devs](https://github.com/greenlsi/bloodtwin_devs), which is the maintained simulator. Check also the [hemovigilance](https://github.com/greenlsi/hemovigilance) repository.
- Pablo Ramos: [Bachelor Thesis](https://youtu.be/H48Uh2r7KfQ) on UI interface in Java and database. The original source code (in Java) is in the folder [hemoglobului of this repository](https://github.com/greenlsi/hemoglobulab_MSO/tree/master/hemoglobului/java).
- Andrés Moreno: [Bachelor Thesis](https://youtu.be/hqC6O5XZ8tI) on app development for Android and C code for hemoglobulin monitoring device. The Android app is in the [hemoglobulab_app](https://github.com/greenlsi/hemoglobulab_app) repository. The C code and the Python code for the data processing of the device were in the now archived [hemoglobulab_device](https://github.com/greenlsi/hemoglobulab_device) repository, and have since been split into the active repositories listed in [Where the device work lives now](#where-the-device-work-lives-now). The app communicated with the device via Bluetooth and sent the data to a server, whose code is in [hemoglobulapp_serverside](https://github.com/greenlsi/hemoglobulapp_serverside); that role is now played by [bloodtwin_webapp_backend](https://github.com/greenlsi/bloodtwin_webapp_backend).
- Esther Moreno: [Bachelor Thesis](2022-estherMoreno.pdf) on the design and development of the miniaturization of the hemoglobulin monitoring device. The annex of the thesis contains the detailed schematics and design documentation of the miniaturized device. The manufacturing outputs of this work — the Gerber and drill files for board revision `PCB_TFG_1.2` and the bills of materials — are in the [bloodtwin_device_hw](https://github.com/greenlsi/bloodtwin_device_hw) repository, which is the authoritative location for the hardware of the device and the starting point for any new board revision. The firmware that runs on this hardware is in [bloodtwin_device_fw](https://github.com/greenlsi/bloodtwin_device_fw).
- Bilal Boulibya: [Master Thesis](2026-bilalBoulibya-TFM.pdf) on the new MAX30102 firmware and its integration with BloodTwin. The reference firmware is in [bloodtwin_device_fw](https://github.com/greenlsi/bloodtwin_device_fw) on `main`: it targets the nRF52840 with Zephyr / nRF Connect SDK, transmits PPG samples through BLE using Protocol Buffers, models its control logic with a verified Petri net, and provides diagnostic tracing and error-recovery tools. The previous nRF5 SDK firmware from Andrés Moreno is preserved in the same repository under `legacy/nrf5-sdk-firmware`.
- Jorge Pérez: TFM on the BloodTwin DEVS digital twin, including adverse reactions and the DEVS–OPL workflow. The maintained implementation is split between [bloodtwin_devs](https://github.com/greenlsi/bloodtwin_devs) and [bloodtwin_scheduler_milp](https://github.com/greenlsi/bloodtwin_scheduler_milp). The final TFM PDF is **pending**: it was not present in `~/Drive/proyectos/hemo/data/turnos_enfermeria_2026` (which contains the October 2026 nursing roster workbook), nor in the dedicated code repository. Add the PDF here when located.
- Eduardo Fernández: Master Thesis on the development of MILP models for the optimization of blood-collection scheduling. The original model is preserved in the archived [hemoglobulab_MSO/hemoglobulopt](https://github.com/greenlsi/hemoglobulab_MSO/tree/master/hemoglobulopt) repository. Eduardo Abreu and Josué Pagán later updated it in the archived [hemoglobulab_opt](https://github.com/greenlsi/hemoglobulab_opt) repository. Its four 2023 thesis experiments are on `master`; the auxiliary local workspace is preserved in `archive/local-2026-09-29` and tagged `archive/eduardo-abreu-experiments-2023`. The maintained model used by BloodTwin is now [bloodtwin_scheduler_milp](https://github.com/greenlsi/bloodtwin_scheduler_milp).
- Alfonso Escribano: [Bachelor Thesis](https://youtu.be/LXsRxujemeU) on the development of a smartphone app with gamification for the donor. It uses Flutter. The app has never been tested. The original source code is in the[hemoglobulab_donors_app](https://github.com/greenlsi/hemoglobulab_donors_app) repository.
- Bernardo Cánovas: Master Thesis on the prediction of hemoglobin levels in donors using regression and classification models. The thesis was written by a student from the University of Murcia. The original source code is not available.

## Publications

Posters, abstracts and papers are indexed in [publicaciones/README.md](../publicaciones/README.md).
