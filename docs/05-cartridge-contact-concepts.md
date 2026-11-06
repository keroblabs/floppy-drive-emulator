# 3D Cartridge PCB Contact Concepts

## Design target

Reliable, repeatable contact between:

- cartridge PCB pads (moving part)
- emulator-side mating contacts (fixed in chassis)

## Candidate contact methods

1. **Spring pogo pins (recommended baseline)**
   - Pros: alignment tolerance, serviceability, easy prototyping
   - Cons: wear on pad finish; needs controlled landing geometry
2. **Leaf spring contacts**
   - Pros: low cost in volume, compact
   - Cons: more sensitive to mechanical tolerance stack-up
3. **Board-edge card connector style**
   - Pros: mature contact architecture
   - Cons: requires stricter insertion alignment and edge plating strategy

## Practical recommendations

- Use gold-finished cartridge pads (ENIG) for durability.
- Add chamfered lead-in features in printed parts for self-centering.
- Place at least two mechanical datum features independent of electrical pads.
- Include a short pre-contact ground-first pad if possible.
- Keep SPI flash and contact routing protected from insertion-force stress.

## Suggested signal grouping on contacts

- Power and ground first
- SPI bus (CS/CLK/MOSI/MISO)
- Presence detect and cartridge ID
- Optional write-protect or media profile pin
