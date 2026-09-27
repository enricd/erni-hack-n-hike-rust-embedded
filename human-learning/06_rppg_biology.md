# 06 — The biology: why a camera can see your pulse

The method is called **rPPG**: remote photoplethysmography. "Plethysmo" =
measuring volume changes, "photo" = with light, "remote" = without
touching the skin.

## 1. Every heartbeat pushes blood into the skin

- The heart pumps in beats. Each beat sends a pressure wave through the
  arteries.
- In the skin, the small arteries (arterioles) and the capillaries fill a
  little more with each wave, then relax.
- So the **amount of blood in the skin rises and falls once per beat**. This
  is the *blood volume pulse*.

A normal resting adult heart rate is about 60–100 beats per minute. Our
search range, 40–180 BPM, covers rest, sport and athletes' slow hearts.

## 2. Blood absorbs green light

Blood is red because **hemoglobin** (the oxygen carrier in red blood cells)
absorbs blue and green light strongly and red light weakly: the red is what
bounces back.

- Oxygenated hemoglobin has strong absorption peaks in the green–yellow,
  around 540 and 575 nm, and a much stronger one in the blue–violet.
- Red light (~650 nm and above) is absorbed far less.

More blood in the skin → more green light absorbed → **the skin looks a
tiny bit darker in the green channel on each beat**.

## 3. Why green and not blue or red?

- **Blue** is absorbed strongly too, but it barely enters the skin, and
  camera sensors are noisy in blue.
- **Red** goes deep, but blood hardly absorbs it, so the pulse is weak.
- **Green** is the sweet spot: absorbed strongly by blood, and it reaches
  the upper skin layers where the capillaries are (roughly the first
  millimetre).
- Bonus for us: RGB565 gives green 6 bits, red and blue only 5.

That is why smartwatches and fitness bands shine **green LEDs** into the
wrist to measure heart rate. (Pulse oximeters for blood oxygen use red and
infrared instead: they compare two colours that oxygenated and
deoxygenated blood absorb differently.)

## 4. How small the signal is

The pulse changes the skin's brightness by **far less than 1%**. In our
numbers: the green mean is ~30 (of 63), and the pulse moves it by a few
tenths of a level. Things that are **much** bigger:

| Disturbance | Why | What we did |
| --- | --- | --- |
| Auto exposure / white balance | the camera "fixes" the brightness | locked them (step 3) |
| Head movement | other skin, other shadows enter the view | hold still; face detection later |
| Light changes | clouds, people walking by | steady light; the line fit removes slow drifts |
| Flickering lamps | 50/100 Hz mains flicker can alias into the band | daylight or good LED light |

Skin tone matters too: **melanin** also absorbs green light, so on darker
skin less light comes back and the pulse is weaker. This is a known
fairness problem of PPG sensors, in watches as well as cameras.

## 5. The research behind it

- **Verkruysse, Svaasand, Nelson (2008)**, *Optics Express*: pulse measured
  from the face with an ordinary camera and ambient light; the **green**
  channel carried the strongest signal. This is exactly what we built.
- **Poh, McDuff, Picard (2010)**, *Optics Express*: webcam heart rate using
  all three colour channels and independent component analysis (ICA).
- **de Haan & Jeanne (2013, "CHROM")** and **Wang et al. (2017, "POS")**,
  *IEEE Transactions on Biomedical Engineering*: combine the colour channels
  so that movement and lighting cancel out. Much more robust than green
  alone, and still cheap enough for a microcontroller: a natural next step.

## 6. Not a medical device

This is a demo. It is not validated, not calibrated, and it fails with
movement, bad light or a partly visible face. Never use it for health
decisions.
