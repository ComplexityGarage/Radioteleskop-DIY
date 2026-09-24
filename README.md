# Radioteleskop DIY 
# Authors 
- Franciszek Solski 
  
# Project objective
  The aim of the project was to construct a simple radio telescope capable of detecting radio emission
from the Sun. The instrument was assembled from components used in satellite television systems
and a software-defined RTL-SDR receiver. The system does not produce an image of the Sun;
instead, it measures the total received power within a selected frequency. In this application,
it works as a simple radiometer.
The primary objective was to determine whether pointing the antenna at the Sun caused an
increase in the received power relative to a nearby region of the sky. A second measurement was
then carried out with the antenna kept stationary, allowing the motion of the Sun across
the sky to move it through the antenna beam. 
# Operating principle
  The Sun emits electromagnetic radiation across a broad spectrum, including radio waves. A metallic
parabolic dish reflects incoming waves towards its focal point, where an LNB converter is mounted.
This component amplifies the signal and converts it to a lower intermediate frequency that can be
received by the RTL-SDR.
The sensitivity of the antenna depends on the direction from which the radiation arrives. The
approximate width of its main beam is described by
θ ∼λ/D

where λ is the wavelength and D is the diameter of the dish. As the Sun approaches the antenna
, its contribution to the received power should increase, reach a maximum, and then decrease as
it moves away from the axis.
# Components used 
The following components were used to construct the system:
• a satellite dish,
• an LNB converter mounted at the focal point of the dish,
• an analogue satellite signal meter (satfinder),
• an external AZS-02 unit supplying the LNB with 12 V,
• an RTL-SDR receiver,
• a computer running macOS,
• librtlsdr, rtl_power, and GQRX software, and a script written in Python.

The dish selected the observed region of the sky. The satfinder provided a quick analogue
indication of changes in signal level, whereas the RTL-SDR enabled digital data to be recorded
for plotting. The frequency selected in the software corresponded to the signal after
conversion in the LNB.
# Construction and system start-up
The dish was mounted on a stand that allowed its azimuth and elevation to be adjusted. The LNB
was attached to the original support arm at the focal point of the antenna. The components of the
signal path were connected using coaxial cables. Power was also supplied to the LNB, as required
for its amplifier to operate. The antenna signal was fed to the RTL-SDR and then transferred to the
computer through a USB connection.
After start-up, correct reception was verified using GQRX. The actual power measurements
were made with rtl_power. A constant receiver gain was used for all recordings because
automatic gain control could compensate for the increase in signal caused by pointing the
antenna towards the Sun. 
# Directional test<img width="994" height="496" alt="pomiar" src="https://github.com/user-attachments/assets/34e58f7b-0118-43bd-a9de-392e51b73dc3" />
 
Figure 1 shows the result of the test in which the antenna was moved manually. The sequence
of pointings was: sky region, Sun, sky region, and Sun again. In both intervals, 
an increase of approximately 0.2–0.25 dB was recorded relative to the nearby sky. The
repeatability of the change after the antenna was pointed at the Sun for a second time indicates
that the system responded to the observing. The steep edges of the curve resulted from
the manual rotation of the antenna and do not represent the natural motion of the Sun.
<img width="812" height="408" alt="test" src="https://github.com/user-attachments/assets/8c8637d7-5cf7-48e9-b1b5-80e27be2e043" />

# Measurement with the antenna stationary
During the measurement presented in Figure 2, the antenna remained stationary. A several-minute
increase in signal level is visible, with the maximum occurring at approximately 13:49. The increase
relative to the local level before the maximum was approximately 0.07–0.08 dB. The timing and
duration of the enhancement are consistent with the position of the sun.
The curve is not perfectly symmetrical. A slow drift in the background level was superimposed
on the useful signal: the power initially decreased, and after the transit it reached values below the
adopted reference level before partially increasing again. Possible causes include thermal stabilisation
of the LNB, satfinder, and RTL-SDR, variations in the gain of the signal path, and interference from
the surroundings. Cloud cover also  affected the measurement. 
<img width="994" height="496" alt="pomiar" src="https://github.com/user-attachments/assets/a7a210cd-e19f-478f-acc5-3ff97f034c05" />


# Summary
The constructed system made it possible to detect a directional change in the received radio power.
In the control test, pointing the antenna at the Sun twice produced a repeatable increase in signal,
whereas pointing it at a nearby region of the sky caused the signal to return to a lower level. During
the measurement with the antenna stationary, a several-minute maximum consistent with the Sun
passing through the antenna beam was recorded. The two results complement one another and
support the interpretation that solar radio emission was detected.
The measurement was qualitative. The system was not calibrated using a source with a known
noise temperature, so the result cannot be converted directly into the solar radio flux. The visible
background drift also demonstrates that the thermal stability of the entire signal path is important
for longer observations. Possible future improvements include allowing the equipment to warm up
before recording, performing several control measurements of empty sky, determining and removing
the slowly varying background, and repeating the solar transit measurement on subsequent days.

<details>
<summary><b>Python code</b></summary>

```python
import csv
import sys
from collections import defaultdict
from datetime import datetime
from pathlib import Path

import numpy as np
import matplotlib.pyplot as plt
import matplotlib.dates as mdates



if len(sys.argv) < 2:
    print("Użycie: python3 drift.py nazwa_pliku.csv")
    sys.exit(1)

plik_wejsciowy = sys.argv[1]
plik_wykresu = str(Path(plik_wejsciowy).with_suffix("")) + "_plot.png"

dane = defaultdict(list)


# Wczytanie z rtl_power
with open(plik_wejsciowy, newline="") as plik:
    for wiersz in csv.reader(plik):

        if len(wiersz) < 7:
            continue

        try:
            czas = datetime.strptime(
                wiersz[0].strip() + " " + wiersz[1].strip(),
                "%Y-%m-%d %H:%M:%S"
            )

            wartosci_db = np.array([
                float(x) for x in wiersz[6:] if x.strip()
            ])

            # Zamiana wartości dB na skalę liniową
            moc_liniowa = 10 ** (wartosci_db / 10)

            dane[czas].extend(moc_liniowa)

        except ValueError:
            continue


if not dane:
    print("Nie znaleziono poprawnych danych w pliku.")
    sys.exit(1)



czasy = sorted(dane.keys())

# Uśrednienie mocy
moc_db = np.array([
    10 * np.log10(np.mean(dane[czas]))
    for czas in czasy
])


#Poziom tła z pierwszych 20% pomiaru
liczba_punktow_tla = max(3, len(moc_db) // 5)
poziom_tla = np.median(moc_db[:liczba_punktow_tla])


moc_wzgledna = moc_db - poziom_tla


#Wygładzanie
liczba_usrednianych_punktow = 5

if len(moc_wzgledna) >= liczba_usrednianych_punktow:
    margines = liczba_usrednianych_punktow // 2

    wygladzona = np.convolve(
        np.pad(moc_wzgledna, (margines, margines), mode="edge"),
        np.ones(liczba_usrednianych_punktow)
        / liczba_usrednianych_punktow,
        mode="valid"
    )
else:
    wygladzona = moc_wzgledna


#Tworzenie wykresu
fig, ax = plt.subplots(figsize=(10, 5))

ax.plot(
    czasy,
    moc_wzgledna,
    color="cornflowerblue",
    alpha=0.45,
    label="Dane surowe"
)

ax.plot(
    czasy,
    wygladzona,
    color="darkblue",
    linewidth=2,
    label="Dane wygładzone"
)

ax.axhline(
    0,
    color="black",
    linewidth=1,
    linestyle="--",
    alpha=0.5
)

ax.set_xlabel("Czas")
ax.set_ylabel("Zmiana mocy względem tła [dB]")
ax.set_title("Zmiana mocy sygnału w czasie")

ax.xaxis.set_major_formatter(
    mdates.DateFormatter("%H:%M:%S")
)

ax.grid(True, alpha=0.3)
ax.legend()

fig.autofmt_xdate()
plt.tight_layout()

plt.savefig(plik_wykresu, dpi=200)
plt.show()

print(f"Wykres zapisano jako: {plik_wykresu}")
```

</details>

