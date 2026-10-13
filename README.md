# mrsim-protocol

An MRI acquisition protocol for building simulator inputs from one human subject.

The simulators are [aslscan](https://github.com/PennLINC/aslscan) (arterial spin labeling) and
[TRXScan](https://github.com/PennLINC/TRXScan) (diffusion), both built on
[mrsim-acq](https://github.com/PennLINC/mrsim-acq). Each takes maps of tissue properties as
input. This protocol describes how to measure those maps on a real person, so that both
simulators can run on the same anatomy.

The protocol is in [protocol.md](protocol.md). It is a proposal: no data has been acquired with
it yet.

## What it covers

1. What the resulting "ground truth" is and is not.
2. The acquisitions: core scans, cheap calibration extras, optional scans, and validation scans.
3. Which simulator input each acquisition feeds today.
4. Simulator extensions the data would motivate, with links to their issues.
5. How to process the scans into an aslscan phantom and TRXScan inputs.
6. A two-session plan.
7. Open questions.

## Status

- Statements about the simulators were checked against specific commits, listed at the top of
  the protocol. They can go stale as the simulators change.
- Scan times are rough estimates and depend on the scanner.
- Several items are undecided and are listed under "Open questions" in the protocol.

## Related issues

Extensions proposed in the protocol are tracked in the simulator repositories:

- mrsim-acq: [#1](https://github.com/PennLINC/mrsim-acq/issues/1),
  [#2](https://github.com/PennLINC/mrsim-acq/issues/2)
- aslscan: [#1](https://github.com/PennLINC/aslscan/issues/1),
  [#2](https://github.com/PennLINC/aslscan/issues/2),
  [#3](https://github.com/PennLINC/aslscan/issues/3),
  [#4](https://github.com/PennLINC/aslscan/issues/4)
- TRXScan: [#15](https://github.com/PennLINC/TRXScan/issues/15),
  [#16](https://github.com/PennLINC/TRXScan/issues/16),
  [#17](https://github.com/PennLINC/TRXScan/issues/17),
  [#18](https://github.com/PennLINC/TRXScan/issues/18),
  [#19](https://github.com/PennLINC/TRXScan/issues/19)

## License

MIT. See [LICENSE](LICENSE).
