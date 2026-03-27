# Speaker Notes

## Slide 1: Hero — "Computer Vision Technology Overview"

Zenna Health has built a complete computer vision pipeline that runs natively on iPhone. Three numbers to set the scene: it processes camera frames at 30 frames per second, it measures joint positions in true 3D — not estimated from a flat image — and everything runs entirely on the device. No frames leave the phone, no cloud round-trips, no internet connection required.

## Slide 2: The Pose Detection Pipeline

Here's how it works, end to end. Every frame from the iPhone's front camera goes through six stages.

First, we capture at 720p in portrait orientation. The frame goes straight to Apple's Vision framework, which is the core of the system — it runs two detection requests simultaneously. One gives us 2D joint positions with confidence scores for each joint. The other gives us full 3D positions in world coordinates, measured in metres, relative to the hip.

From those detections we build a 20-point skeleton. Apple Vision gives us 16 joints natively — head, shoulders, elbows, wrists, hips, knees, ankles. We synthesise four additional foot joints — heels and toes — from the ankle and knee geometry, because ankle angles are critical for lower-limb rehab.

The raw landmark data is noisy, so we apply a One Euro filter — it's an adaptive low-pass filter that smooths heavily when the patient is still, but backs off when they move fast. This cuts jitter by roughly 70% without adding lag.

From the smoothed 3D coordinates we compute nine clinically-relevant joint angles — I'll show those on the next slide. And finally, all of that feeds into the exercise engine, which counts reps, scores form, and generates corrective feedback. All six stages complete in under 14 milliseconds per frame.

## Slide 3: Joint Angle Measurement

These are the nine angles we measure. They're computed from true 3D vectors — the formula is a standard dot-product angle calculation, but the key point is that the inputs are world coordinates in metres, not pixel positions projected onto a 2D image. That means the angles are accurate regardless of where the patient stands relative to the camera.

For lower-limb rehab, the most important ones are knee flexion, hip flexion, and hip abduction. We also measure ankle dorsiflexion and plantarflexion — that's where the synthesised foot joints come in. Shoulder and elbow flexion cover upper body exercises, and trunk lean measures whether the patient is compensating by leaning.

Each angle is defined as a triple of joints — for example, knee flexion is the angle at the knee formed by the hip, knee, and ankle. The system knows which joints to use for left-side versus right-side exercises, and it mirrors them automatically for bilateral exercises.

## Slide 4: How Form Scoring Works

For every frame, we compare the patient's joint angles against a reference pose — recorded from a demonstration video of correct form. The scoring uses a Gaussian decay model.

If the patient's angle matches the reference exactly, they score 100. As the error increases, the score drops off following a bell curve with sigma of 15 degrees. At 10 degrees of error you're still scoring around 80% — that's the green zone, good form. Between 10 and 20 degrees you're in the yellow zone, and beyond 20 degrees you're in red, which triggers a corrective cue.

The joints are weighted per exercise. For a squat, knee flexion carries 40% of the score, hip flexion 30%, and trunk lean 30%. These weights are configurable per exercise in JSON — they're not hardcoded.

The raw per-frame score is smoothed with an exponential moving average so the display doesn't flicker. And the skeleton overlay on screen is colour-coded — each joint lights up green, yellow, or red based on its individual score, so the patient can see exactly which joint needs correction.

## Slide 5: How Rep Counting Works

Rep counting uses a state machine. Each exercise defines a set of states with angle thresholds. For a squat, standing is 150 to 180 degrees of knee flexion, descending is 140 to 150, and bottom is below 140. A rep is counted when the patient completes the full cycle — standing to bottom and back to standing.

Two things prevent false counts. First, hysteresis — there's a plus-or-minus 3 degree deadband at each threshold, so if the patient's knee is oscillating right at 150 degrees, the state doesn't flip back and forth. Second, a minimum rep duration — typically 0.4 to 1 second depending on the exercise — so a brief jitter can't register as a complete rep.

The system also shows a ghost skeleton — a semi-transparent reference pose overlaid on the camera feed. It's not just playing back a video; it finds the reference frame whose primary angle is closest to the patient's current angle, so the ghost is always showing what correct form looks like at that exact point in the movement.

Corrective feedback fires from configurable rules — for example, if knee flexion is above 145 degrees while the state machine says you should be in the bottom position, it shows "Bend your knees more". Messages are debounced at 3 seconds so patients aren't overwhelmed.

## Slide 6: Kemtai vs Zenna

This is a straight technical comparison. Kemtai runs in a web browser using TensorFlow.js for pose estimation. It produces 2D joint positions — depth is inferred, not measured. Processing involves a cloud round-trip, which adds 200 to 500 milliseconds of latency and means it doesn't work offline.

Zenna Health's solution runs natively on iOS using Apple's Vision framework, which is hardware-accelerated on the Neural Engine. It produces true 3D world coordinates. Processing is entirely on-device — under 14 milliseconds, works offline, and no patient video data ever leaves the phone.

Range of motion tracking is built into every tracked exercise automatically — Kemtai treats it as a separate assessment mode. And critically, all the data — exercises, videos, patient records, session history — lives on Zenna Health's own infrastructure.

## Slide 7: What We've Migrated

This slide shows the current state of the migration away from Kemtai. The exercise library — all 1,506 exercises — has been seeded into Zenna Health's own Supabase database with full metadata. The exercise demonstration videos have been downloaded from Kemtai's CloudFront CDN and re-hosted on Vercel Blob storage in both MP4 and WebM formats.

Patient data and protocol management, which previously went through Kemtai's REST API, now runs against Zenna Health's own database with row-level security. Staff profiles, patient records, pathways, session history — all native.

The Kemtai-specific database tables — kemtai_staff_links and kemtai_patient_profiles — were dropped on March 27th. There are no remaining technical dependencies on Kemtai's platform.

## Slide 8: Tracked Exercises & Expansion

Today, six exercises are fully tracked with camera-based form scoring, rep counting, and corrective feedback. These are ACL rehabilitation exercises — squats, heel raises, knee extensions, and jump rope.

The full library of 1,506 exercises is already in the database with videos stored. The important thing is that the exercise engine is entirely data-driven. Adding camera tracking to a new exercise doesn't require any code changes. You record a reference video, run it through a Python pipeline that extracts pose landmarks and segments reps automatically, and add a JSON configuration file that defines the angle thresholds, state machine, and scoring weights.

The entire computer vision pipeline is roughly 2,000 lines of Swift code. It requires iOS 17 or later for the 3D pose detection API, which covers all iPhones from the 2018 XR onwards.

## Slide 9: Research Informing the Design

The design decisions behind this system are grounded in peer-reviewed research. I'll highlight a few key ones.

The clinical validation paper from PMC in 2024 showed that pose estimation achieves 5 to 10 degrees of mean error against goniometry for major joints, with correlation above 0.85. That's not precise enough for clinical measurement, but it's well within range for exercise guidance and wellness feedback — which is how this system is positioned under the FDA's General Wellness Policy.

The Pose Trainer paper from 2020 found that angle-based heuristic rules generate better corrective feedback than trying to classify form errors using DTW. That's why the feedback system uses simple threshold rules — "if knee flexion is above 145 degrees at the bottom of a squat, show 'bend your knees more'" — rather than trying to do pattern matching on the full movement sequence.

Two papers from Nature Scientific Reports in 2025 validated DTW-based scoring for physiotherapy. One showed that adding per-joint weights to DTW improves scoring accuracy by 12 to 18 percent, which is why each exercise configuration has configurable joint weights.

The telerehab adherence review from the Journal of Physiotherapy is particularly relevant for the business case — camera-guided physiotherapy showed 85% patient adherence compared to 62% for paper-based programmes.

And the One Euro Filter paper underpins the landmark smoothing — it's a well-established adaptive filter that reduces jitter when the patient is still but preserves responsiveness during fast movement.

## Slide 10: Closing — Summary

To summarise: Zenna Health has a production-grade, on-device computer vision system for rehabilitation exercise tracking. It uses Apple's Vision framework for true 3D pose detection, runs at 30 frames per second with under 14 milliseconds of latency, and the full Kemtai exercise library has been migrated to Zenna Health's own infrastructure. The system is designed for data-driven expansion — new exercises can be added without code changes.
