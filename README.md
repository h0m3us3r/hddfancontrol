# HDD Fan control

[![PyPI version](https://img.shields.io/pypi/v/hddfancontrol.svg?style=flat)](https://pypi.python.org/pypi/hddfancontrol/)
[![AUR version](https://img.shields.io/aur/version/hddfancontrol.svg?style=flat)](https://aur.archlinux.org/packages/hddfancontrol/)
[![Tests status](https://img.shields.io/travis/desbma/hddfancontrol/master.svg?label=tests&style=flat)](https://travis-ci.org/desbma/hddfancontrol)
[![Coverage](https://img.shields.io/coveralls/desbma/hddfancontrol/master.svg?style=flat)](https://coveralls.io/github/desbma/hddfancontrol?branch=master)
[![Lines of code](https://tokei.rs/b1/github/desbma/hddfancontrol)](https://github.com/desbma/hddfancontrol)
[![Supported Python versions](https://img.shields.io/pypi/pyversions/hddfancontrol.svg?style=flat)](https://pypi.python.org/pypi/hddfancontrol/)
[![License](https://img.shields.io/github/license/desbma/hddfancontrol.svg?style=flat)](https://github.com/desbma/hddfancontrol/blob/master/LICENSE)

HDD Fan control is a command line tool to dynamically control fan speed according to hard drive temperature on Linux.

This has 3 benefits:

- it allows maintaining your hard drives in the ideal temperature range to have maximum longevity and avoid overheating

Because fans will slow down or stop when not needed:

- it minimizes noise generated by the fans
- it also minimizes power consumption at the same time

## When is this useful?

HDD Fan control is useful when you have one or several hard drives with one or several fans close to them, and do not want to let the motherboard control the fan speed, because it does so either statically, or using a temperature sensor unrelated to the real drive temperature (either on the CPU or on some other place on the motherboard).

The ideal use case is for a NAS with several hard drives, a low power CPU (ie. ARM or Intel Atom) with passive cooling (no fans), and a chassis with fans close to the hard drive. It that case the CPU will generate less heat than the hard drives and it makes sense to control fan speed according to the main heat source.

## Features

- Can run in daemon mode
- Can control several fans and/or several drives with a single invocation
- Can automatically spin down drives after a customizable period of inactivity
- Can adapt to different fan characteristics
- Can be set to stop fans or run them at full speed at customizable temperatures
- Can be configured to never set the fans below a certain speed (useful if the fans controlled by HDD Fan control are the only ones available in the chassis)
- Can also optionally monitor CPU temperature, and control fan speed accordingly

## Prerequisites

- A Linux distribution
- A least one SATA hard drive, that supports:
  - Temperature querying
  - Power state querying
- A motherboard which:
  - exposes to the OS a PWM to control fan speed
  - exposes to the OS a sensor to query fan speed

Most motherboards and SATA drives fit these requirements.

## Installation

HDD Fan control requires [Python](https://www.python.org/downloads/) >= 3.4.

### Distribution packages

Some Linux distributions have a hddfancontrol package available in their repository:

- Arch Linux (AUR): [hddfancontrol](https://aur.archlinux.org/packages/hddfancontrol/)
- Fedora: [hddfancontrol](https://apps.fedoraproject.org/packages/hddfancontrol) (thanks to [TC01](https://github.com/TC01))

### From PyPI (with PIP)

1. If you don't already have it, [install pip](https://pip.pypa.io/en/stable/installing/) for Python 3
2. Install HDD Fan control: `pip3 install hddfancontrol`
3. Install [hdparm](http://sourceforge.net/projects/hdparm/) and [hddtemp](http://www.guzu.net/linux/hddtemp.php).
   On Ubuntu and other Debian derivatives: `sudo apt-get install hdparm hddtemp`.

### From source

1. If you don't already have it, [install setuptools](https://pypi.python.org/pypi/setuptools#installation-instructions) for Python 3
2. Clone this repository: `git clone https://github.com/desbma/hddfancontrol`
3. Install HDD Fan control: `python3 setup.py install`
4. Install [hdparm](http://sourceforge.net/projects/hdparm/) and [hddtemp](http://www.guzu.net/linux/hddtemp.php).
   On Ubuntu and other Debian derivatives: `sudo apt-get install hdparm hddtemp`.

To query fan characteristics, you may also need pwmconfig. On Ubuntu and other Debian derivatives, it is part of the fancontrol package, that you can install with `sudo apt-get install fancontrol`. HDD fancontrol and fancontrol are unrelated. The fancontrol daemon is **not** needed for HDD fan control to operate. If you use both fancontrol and HDD fancontrol, be careful not to make them control the same fans.

## Configuration

### A word of caution

The default parameters will run fans at 100% speed at temperatures > 50°C, and run them a 20% speed if < 30°C, which corresponds to the usual recommended drive operating temperature. If you are sure that there are no other components in your system that generate significant heat, if you have other fans to cool down youy system, or if you have a case optimized for passive cooling, you can set minimum speed to 0%, which will stop the fans if temperature is below the minimum threshold.

**Be aware that a misconfiguration of this tool can lead to a failure to cool down your system properly which can damage components or reduce their lifetime.**

Before using HDD Fan control unmonitored for long period of time, I recommend keeping a minimum fan speed for security, and checking that the temperature of your system stays in reasonable range as expected.

### Fan configuration

To get the value for the `--pwm`, `--pwm-start-value` and `--pwm-stop-value` parameters, you can either:

- Use the `-t` or `--test` parameter, which will run some tests and detect the values at which the fans start and stop. However you need to have previously identified the PWM file (the `--pwm` parameter)
- use the [pwmconfig tool](http://www.lm-sensors.org/wiki/man/pwmconfig).

### Drive auto spin down

SATA drives can be configured to automatically spin down after a certain period of inactivity, which saves power. If your drives are configured to do so, you may notice that they do not spin down when HDD Fan control is running.
This is due to the fact that HDD Fan control will query temperature at fixed interval, which the drive will consider an activity and reset the spin down timeout.
To fix that, you can either:

- Use a value for the `-i`/`--interval` parameter higher than your SATA spin down time (not recommended unless your spin down time is very low, ie < 2 min). In that case you do not need to use hddfancontrol's `--spin-down-time` switch, because the builtin SATA drive timeout (that you can set for example with [`hdparm -S XXX` command](https://linux.die.net/man/8/hdparm)) should take effect.
- Use the `--spin-down-time` parameter that will monitor drive activity and spin it down if inactive, independantly of the SATA feature (recommended)

**Keep in mind that spinning down and up a drive repeatedly wears it prematurly, so unless you are in a power constrained environement (ie. laptop), do not set the spin down time too low.**

Reading temperature while a drive is in low power state will make it spin up, so HDD Fan control will stop querying temperature in that case, and wait for the drive (which will be cooling down in low power state anyway) to spin up.

Some HGST (previously Hitachi) drives support a special way of querying temperature that does not spin up drives, which HDD Fan control will detect and use, however it still prevents them from spinning down, so the above instructions still apply.

## Command line usage

Run `hddfancontrol -h` to get full command line reference.

As an example, the command line below will instruct HDD Fan control to:

- monitor temperature of drives `/dev/sda` and `/dev/sdb`
- control fan speed using PWM 2 and 3 in `/sys/class/hwmon/hwmon1/device/`
- start both fans using PWM value 200
- consider the fans will stop with PWM value 75
- never run the fans below 10% of their maximum speed
- check temperature at least every minute
- automatically spin down drives if they are inactive for 2 hours (7200 seconds)
- run in daemon mode
- log what is going on to `/var/log/hddfancontrol.log`

`hddfancontrol -d /dev/sda /dev/sdb -p /sys/class/hwmon/hwmon1/device/pwm2 /sys/class/hwmon/hwmon1/device/pwm3 --pwm-start-value 200 200 --pwm-stop-value 75 75 --min-fan-speed-prct 10 -i 60 --spin-down-time 7200 -b -l /var/log/hddfancontrol.log`

## Systemd service

A systemd service file is provided to control the daemon easily.
If you installed hddfancontrol from a distribution package, you likely already have it installed, otherwise you can install it from the sources of this repository with:

```
sudo cp ./systemd/hddfancontrol.service /etc/systemd/system/
sudo mkdir -p /etc/conf.d
sudo cp ./systemd/hddfancontrol.conf /etc/conf.d/hddfancontrol
```

Then you need to edit the `HDDFANCONTROL_ARGS` variable in `/etc/conf.d/hddfancontrol` to set the parameters (drives, temperature range...).

You can then start the daemon with `sudo systemctl start hddfancontrol`, see its status with `sudo systemctl status hddfancontrol` and enable automatic startup at boot time with `sudo systemctl enable hddfancontrol`.

## License

[GPLv3](https://www.gnu.org/licenses/gpl-3.0-standalone.html)
