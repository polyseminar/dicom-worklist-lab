# DICOM worklist lab

Follow one CT exam through a radiology department on your own computer, for learning: the RIS sends an HL7 order, the order becomes a DICOM modality worklist entry, the scanner asks for its worklist, writes the patient and the accession number into its images, and sends them to the PACS. The lab runs an [Orthanc](https://www.orthanc-server.com/) PACS with its worklists plugin, and [DCMTK](https://dcmtk.org), the OFFIS DICOM command-line tools, as the scanner.

This lab goes with the fifth Polyseminar lab video, on the radiology workflow. It follows [`pacs-lab`](https://github.com/polyseminar/pacs-lab), [`dicom-send-lab`](https://github.com/polyseminar/dicom-send-lab), [`dicom-errors-lab`](https://github.com/polyseminar/dicom-errors-lab) and [`dicom-anonymize-lab`](https://github.com/polyseminar/dicom-anonymize-lab). This one stands alone, so you don't need the others.

> **Lab use only.** Load only the public test data named below. Never load real patient data, including your own scans, and never point these tools at a hospital system without permission. The "RIS" and the "scanner" here are small scripts that show the messages; they are not products.

## What you need

- **Docker**, with at least 8 GB of RAM in the computer:

  | System | What to install |
  | :--- | :--- |
  | Windows 11 | Docker Desktop, then WSL. See [Install Docker on Windows](#install-docker-on-windows) below. |
  | macOS | [Docker Desktop](https://docs.docker.com/desktop/setup/install/mac-install/) for Apple silicon or Intel |
  | Linux | [Docker Engine](https://docs.docker.com/engine/install/) with the Compose plugin |

- **About 3.5 GB of free disk space.** The Orthanc image uses about 2.6 GB, the tools helper about 170 MB, the downloaded study about 220 MB, and the PACS about 250 MB once it holds two scans.

### Install Docker on Windows

1. Open the **Microsoft Store**, search for **Docker Desktop**, and click **Install**. It does not ask for an administrator.
2. Turn on WSL, the Windows Subsystem for Linux, which Docker Desktop runs on. Click Start, type `Terminal`, right-click **Terminal**, and choose **Run as administrator**. Then run:

   ```
   wsl --install --no-distribution
   ```

3. Restart the computer.
4. Start Docker Desktop and wait until it shows **Engine running**.

Docker Desktop is free for personal use, education, and small businesses (fewer than 250 employees and less than $10 million in annual revenue). Larger organizations and government entities need a paid subscription ([Docker Desktop license terms](https://docs.docker.com/subscription-billing/desktop-license/)). On a work computer, ask your IT team before you install it.

## Get the lab

Download **[dicom-worklist-lab.zip](https://github.com/polyseminar/dicom-worklist-lab/releases/latest/download/dicom-worklist-lab.zip)** and extract it. You get one folder, `dicom-worklist-lab`, with `compose.yaml` inside. Open a terminal in that folder:

| System | How |
| :--- | :--- |
| Windows 11 | In File Explorer, right-click the ZIP and choose **Extract All**, then **Extract**. Right-click the `dicom-worklist-lab` folder and choose **Open in Terminal** |
| macOS | Double-click the ZIP. Open Terminal, type `cd` and a space, drag the folder into the window, and press Enter |
| Linux | Extract the ZIP in the file manager, right-click the folder, and choose **Open in Terminal** |

You can also clone the repository. GitHub's own **Code > Download ZIP** works too, but on Windows it puts the folder inside a folder of the same name; use the inner one.

If a lab from an earlier video is running, stop it first with `docker compose down` in its folder: all five labs use ports 8042 and 4242.

## Start the lab

The commands are the same in PowerShell, Command Prompt, macOS Terminal, and Linux shells. Docker Desktop must be running first.

1. Start the PACS:

   ```
   docker compose up -d
   ```

2. Download the test study into Docker:

   ```
   docker compose run --rm download
   ```

   It ends with a list of folders: one collection, one patient, two studies, and one series in each study.

3. Open <http://localhost:8042> and sign in with user `orthanc` and password `orthanc`. The study list is empty, and so is **Worklists** in the menu on the left.

## 1. The order

```
docker compose run --rm tools order
```

The first run builds the tools helper. `order` plays the RIS, the radiology information system. It prints the order the RIS sends, an HL7 version 2 `ORM^O01` message:

```
The RIS sends an order (HL7 ORM^O01):
  MSH|^~\&|RIS|RADIOLOGY|BROKER|RADIOLOGY|202611061400||ORM^O01|202611061400|P|2.5.1
  PID|1||PAN_03||PAN_03|||F
  ORC|NW|A1001|||SC||^^^202611061400
  OBR|1|A1001||CTABDC^CT abdomen with contrast||||||||||||||A1001||||||CT

Worklist entry for CT1: patient PAN_03, accession A1001, CT abdomen with contrast
```

Each line is a segment, and `|` separates its fields. `PID` is the patient, `ORC` the order (`NW`, new), and `OBR` the exam: the procedure code and name, the accession number `A1001`, and the modality `CT`. The accession number is the RIS's number for this order; every system after it uses it to find the order again.

Then `order` does what an HL7-to-DICOM broker does in a hospital: it reads those fields and creates a modality worklist entry on the PACS for the scanner whose AE title is `CT1`. Refresh **Worklists** in the browser to see it.

## 2. The worklist

```
docker compose run --rm tools worklist
```

`worklist` plays the scanner. It sends a DICOM C-FIND to the PACS as `CT1`, asking for every CT procedure step scheduled for that station, with DCMTK's `findscu -W`. The answer is the entry, as DICOM tags: the accession number, the patient's name and ID, a Study Instance UID, and the scheduled step (modality, station, date, description).

On a real scanner, the technologist sees these entries as a list on the console, picks the patient on the table, and the scanner copies the details into every image. Nobody types the name.

## 3. The scan

```
docker compose run --rm tools scan
```

`scan` asks for the worklist again, takes the entry, and makes the images: it copies the study you downloaded, writes the entry's patient, accession number and Study Instance UID into every image, gives them today's date, and sends them to the PACS with C-STORE as `CT1`. It ends with `Sent 226 CT images to the PACS`.

Refresh the study list: one study, patient `PAN_03`, `CT abdomen with contrast`, accession `A1001`. Open it in the viewer, too.

Wait about 15 seconds and refresh **Worklists**: the entry is gone. This PACS deletes an entry once a complete study with its Study Instance UID has arrived (Orthanc counts a study as complete 10 seconds after its last image). That is this lab's shortcut. In a hospital, the scanner reports the step itself with another DICOM service, Modality Performed Procedure Step (MPPS): in progress when it starts, completed when it ends. Images in the PACS don't prove the step was completed.

## 4. Without the worklist

```
docker compose run --rm tools scan PAN03
```

With an ID after it, `scan` skips the worklist and uses the ID as if the technologist had typed it at the console, with the underscore missed and no accession number. Refresh the study list: there are now two patients, `PAN_03` and `PAN03`. The second study matches no order, so the RIS can't find it for the radiologist's report. Someone has to find it and fix it by hand. The worklist exists to prevent this.

## Stop and reset

| Command | Result |
| :--- | :--- |
| `docker compose down` | Stops the PACS. Your studies and worklist entries stay. |
| `docker compose down -v` | Stops the PACS and deletes its studies and worklist entries. The downloaded study stays; run `order` again to start over. |
| `docker compose --profile tools down -v --rmi local` | Deletes everything: the PACS data, the downloaded study, and the tools helper. |

`order` always makes accession `A1001`, and it refuses while that accession is still on the worklist. To run the whole exam again, run `scan` first, or reset with `docker compose down -v`.

## What runs

| Part | Detail |
| :--- | :--- |
| `pacs` | `orthancteam/orthanc:26.9.1` (Orthanc 1.13.0) with Orthanc Explorer 2, the OHIF viewer, DICOMweb, and the worklists plugin (0.9.2), which keeps its entries in the Orthanc database. Web interface on `localhost:8042`, DICOM on `localhost:4242`, AE title `ORTHANC`. It accepts images and worklist queries from any sender |
| `download` helper | Runs the NCI Imaging Data Commons tool `idc-index` 0.12.5 in a temporary container and saves the study in the `study` volume |
| `tools` helper | DCMTK 3.6.9 and Python from Debian 13, built on first use, with the lab's scripts: `order` (the RIS and the broker), `worklist` and `scan` (the scanner, AE title `CT1`). Reads the study read-only and reaches the PACS as `pacs` (DICOM on 4242, the REST API on 8042) |

Both ports are bound to `127.0.0.1`, so only this computer can reach them. The `orthanc` password is acceptable only for that reason. Do not expose this lab to a network.

The settings that make the worklist close itself are in `compose.yaml`: `ORTHANC__WORKLISTS__DELETE_WORKLISTS_ON_STABLE_STUDY`, a 5-second housekeeping interval, and `ORTHANC__STABLE_AGE` of 10 seconds (Orthanc's default is 60).

## Test data

The lab uses patient `PAN_03` from the CTpred-Sunitinib-panNET collection: two contrast-enhanced abdominal CT studies with 422 images in total, from a GE Discovery CT750 HD scanner with 1.25 mm slices. `scan` uses the newer study (226 images) and changes copies of it: the patient's name and ID, the accession number, the study description, the dates, the station name, and new UIDs. `PAN_03` is TCIA's code, not a name.

**Citation:**
Chen, L., Wang, W., Jin, K., Yuan, B., Tan, H., Sun, J., Guo, Y., Luo, Y., Feng, S.-ting, Yu, X., Chen, M.-hu, & Chen, J. (2022). *Prediction of Sunitinib Efficacy using Computed Tomography in Patients with Pancreatic Neuroendocrine Tumors (CTpred-Sunitinib-panNET)* (Version 1) [Data set]. The Cancer Imaging Archive. <https://doi.org/10.7937/SPGK-0P94>

- **License:** [Creative Commons Attribution 4.0 (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). The lab downloads the files unchanged; the images `scan` sends are modified copies, as listed above.
- **Data use:** follow the [TCIA Data Usage Policy](https://www.cancerimagingarchive.net/data-usage-policies-and-restrictions/). Credit the dataset whenever you show it, and do not try to identify or contact the people in it.
- **Download service:** NCI Imaging Data Commons. Fedorov, A., et al. (2023). National Cancer Institute Imaging Data Commons: Toward Transparency, Reproducibility, and Scalability in Imaging Artificial Intelligence. *RadioGraphics*, 43(12). <https://doi.org/10.1148/rg.230180>

## Sources

- DICOM PS3.4, Annex K (Basic Worklist Management Service, Modality Worklist) and Annex F (Modality Performed Procedure Step). <https://www.dicomstandard.org/current>
- IHE Radiology Technical Framework, Scheduled Workflow (SWF.b). <https://www.ihe.net/resources/technical_frameworks/#radiology>
- HL7 Version 2, ORM^O01 general order message.
- Orthanc worklists plugin: <https://orthanc.uclouvain.be/book/plugins/worklists-plugin-new.html>
- DCMTK `findscu`, `dcmodify` and `storescu`: <https://support.dcmtk.org/docs/>

## Troubleshooting

| Problem | Fix |
| :--- | :--- |
| `docker` is not recognized, or `command not found` | Install Docker, then open a new terminal. |
| `no configuration file provided: not found` | The terminal is not in the lab folder. Open it in the folder that contains `compose.yaml`. |
| `Cannot connect to the Docker daemon`, or an error that mentions `dockerDesktopLinuxEngine` | Start Docker Desktop and wait until it shows that the engine is running. On Linux, start the Docker service. |
| `permission denied` on Linux | Put `sudo` in front of the command, or add your user to the `docker` group. |
| `port is already allocated` | Another program uses port 8042 or 4242, often the PACS from an earlier lab. Run `docker compose down` in that lab's folder, or stop the other program. |
| `worklist` prints nothing | The worklist is empty: run `order` first. After a `scan`, the entry closes by itself. |
| `scan` says `The worklist for CT1 is empty` | Run `order` first. |
| `scan` fails at once, before `Sent` | Download the study first (Start the lab, step 2). |
| `order` says `Accession A1001 is already on the worklist` | The entry from an earlier `order` is still open. Run `scan`, or reset with `docker compose down -v`. |
| The entry stays on **Worklists** after `scan` | Wait 15 seconds and refresh. It closes only once the PACS has the whole study. |
| The sign-in prompt keeps returning | Use user `orthanc` and password `orthanc`. |
| The download stops with a network error | Run it again. |

## License

The lab files (`compose.yaml`, this README) are under the [MIT License](LICENSE). The software the lab runs keeps its own license: Orthanc is GPL, the Orthanc worklists plugin is AGPL, DCMTK is BSD-style, OHIF is MIT, and the test data is CC BY 4.0 (see Test data). The Polyseminar videos and course materials are not covered by this license; they are © Polyseminar, all rights reserved.

## Independence

Polyseminar is an independent education publisher. It is not affiliated with or endorsed by a certification body, professional society, healthcare provider, or imaging vendor. DCMTK, Orthanc, OHIF, TCIA, and IDC are the work of their own authors.
