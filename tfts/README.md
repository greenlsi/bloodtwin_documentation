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
| Device firmware | `hemoglobulab_device`, branch `master` (`device-fw/`) | [bloodtwin_device_fw](https://github.com/greenlsi/bloodtwin_device_fw) |
| Signal processing and ML models | `hemoglobulab_device`, branch `devel_modeling` | [bloodtwin_device_models](https://github.com/greenlsi/bloodtwin_device_models) |
| Board design, Gerbers and BOM | never in git, only in Drive | [bloodtwin_device_hw](https://github.com/greenlsi/bloodtwin_device_hw) |
| Android app | `hemoglobulab_device`, branch `devel` (`android-app/`) | **archived only**, superseded by [bloodtwin_webapp_frontend](https://github.com/greenlsi/bloodtwin_webapp_frontend) and [bloodtwin_webapp_backend](https://github.com/greenlsi/bloodtwin_webapp_backend) |

Nothing was deleted. The archived repository keeps its full history, and each branch is
also reachable through a tag: `archive/device-firmware`, `archive/data-modeling` and
`archive/android-app-tfg-amoreno`.

The Android app is intentionally **not** carried over into the new structure. It is
superseded by the web application and is kept purely as a historical record.

Note on the modelling code: the 2023 exploration in `devel_modeling` was left unfinished.
The version of the code that actually produced the published results is the one now in
`bloodtwin_device_models`, under `src/`, with the earlier exploration preserved in
`legacy/`.

## Thesis

- Nacho Uranga: [Master Thesis](https://youtu.be/VVGAXvXanxs) on DEVS simulator. The original source code (in Java) is in the folder [hemoglobulsim of this repository](https://github.com/greenlsi/hemoglobulab_MSO/tree/master/hemoglobulsim). Check also the [hemovigilance](https://github.com/greenlsi/hemovigilance) repository. You can also access temporal and unfinished code of an attempt to migrate the code to DEVS in Python in: [chrm_sim](meet.google.com/fqf-wuxi-hcm). Yo can see a few examples in Python in the `devel` branch in [bdslib](https://github.com/greenlsi/bdslib/tree/devel/).
- Pablo Ramos: [Bachelor Thesis](https://youtu.be/H48Uh2r7KfQ) on UI interface in Java and database. The original source code (in Java) is in the folder [hemoglobului of this repository](https://github.com/greenlsi/hemoglobulab_MSO/tree/master/hemoglobului/java).
- Andrés Moreno: [Bachelor Thesis](https://youtu.be/hqC6O5XZ8tI) on app development for Android and C code for hemoglobulin monitoring device. Both the Android app and the C code were originally in the now archived [hemoglobulab_device](https://github.com/greenlsi/hemoglobulab_device) repository, together with the Python code for the data processing of the device. That work has since been split into the active repositories listed in [Where the device work lives now](#where-the-device-work-lives-now). The app communicated with the device via Bluetooth and sent the data to a server, whose code is in [hemoglobulapp_serverside](https://github.com/greenlsi/hemoglobulapp_serverside); that role is now played by [bloodtwin_webapp_backend](https://github.com/greenlsi/bloodtwin_webapp_backend).
- Esther Moreno: [Bachelor Thesis](2022-estherMoreno.pdf) on the design and development of the miniaturization of the hemoglobulin monitoring device. The annex of the thesis contains the detailed schematics and design documentation of the miniaturized device. The manufacturing outputs of this work (Gerber production files, bill of materials and mechanical design) are kept in the [bloodtwin_device_hw](https://github.com/greenlsi/bloodtwin_device_hw) repository, which is the authoritative location for the hardware of the device. The firmware that runs on this hardware is in [bloodtwin_device_fw](https://github.com/greenlsi/bloodtwin_device_fw).
- Eduardo Fernández: Master Thesis on the development of MILP models for the optimization of the scheduling of blood collection. The original source code is in the folder [hemoglobulopt of this repository](https://github.com/greenlsi/hemoglobulab_MSO/tree/master/hemoglobulopt). This model has been updated and improved in the [hemoglobulab_opt](https://github.com/greenlsi/hemoglobulab_opt/tree/master/hemoglobulopt) repository by Eduardo Abreu and Josué Pagán.
- Alfonso Escribano: [Bachelor Thesis](https://youtu.be/LXsRxujemeU) on the development of a smartphone app with gamification for the donor. It uses Flutter. The app has never been tested. The original source code is in the[hemoglobulab_donors_app](https://github.com/greenlsi/hemoglobulab_donors_app) repository.
- Bernardo Cánovas: Master Thesis on the prediction of hemoglobin levels in donors using regression and classification models. The thesis was written by a student from the University of Murcia. The original source code is not available.
