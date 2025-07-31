# Echopype Bug Report: EK80 Wide Band Data Calibration Issue

## Overview
This repository demonstrates a bug in the echopype library (version 0.10.1) when processing EK80 wide band acoustic data. The bug specifically occurs when attempting to compute volume backscattering strength (Sv) from netCDF files that have been converted from raw EK80 data, then reopened using `ep.open_converted()`.

## Bug Description
The issue manifests as a dimension mismatch error in the xarray library when echopype attempts to calibrate EK80 wide band data that has been:
1. Converted from raw format to netCDF
2. Saved to disk
3. Reopened using `ep.open_converted()`
4. Processed through `ep.calibrate.compute_Sv()`

## Data Specifications
- **Echosounder**: Simrad EK80
- **Data Type**: Wide band (complex samples)
- **File Format**: Raw (.raw) converted to netCDF (.nc)
- **Waveform Mode**: CW (Continuous Wave)
- **Encode Mode**: Complex

## When the Bug Occurs
The bug occurs when:
- Using EK80 wide band data
- Converting raw file to netCDF with `ep.open_raw()` and `to_netcdf()`
- Reopening the netCDF file with `ep.open_converted()`
- Attempting calibration with `ep.calibrate.compute_Sv(echodata, waveform_mode="CW", encode_mode="complex")`

## When the Bug Does NOT Occur
The bug **DOES NOT** occur when:
- Using the original `EchoData` object directly from `ep.open_raw()` for calibration
- Processing EK60 narrowband data (this appears to work correctly)
- Using different data processing workflows that don't involve the save/reload cycle

## Error Details
The calibration fails with the following error:

```
ValueError: dimensions ('channel',) must have the same length as the number of data dimensions, ndim=2
```

### Full Stack Trace
```
ValueError                                Traceback (most recent call last)
Cell In[6], line 2
      1 # Convert echo data from open_converted to sv
----> 2 ds_Sv = ep.calibrate.compute_Sv(echodata, waveform_mode="CW", encode_mode="complex") 

File echopype\calibrate\api.py:208, in compute_Sv(echodata: EchoData, **kwargs)
--> 208     return _compute_cal(cal_type="Sv", echodata=echodata, **kwargs)

File echopype\calibrate\api.py:66, in _compute_cal(cal_type, echodata, env_params, cal_params, ecs_file, waveform_mode, encode_mode)
---> 66     cal_ds = cal_obj.compute_Sv()

File echopype\calibrate\calibrate_ek.py:630, in CalibrateEK80.compute_Sv(self)
--> 630     return self._compute_cal(cal_type="Sv")

File echopype\calibrate\calibrate_ek.py:614, in CalibrateEK80._compute_cal(self, cal_type)
--> 614     ds_cal = self._cal_complex_samples(cal_type=cal_type)

File echopype\calibrate\calibrate_ek.py:548, in CalibrateEK80._cal_complex_samples(self, cal_type)
--> 548 tau_effective[ch_GPT] = beam["transmit_duration_nominal"][ch_GPT].isel(ping_time=0)

File xarray\core\dataarray.py:905, in DataArray.__setitem__(self, key, value)
--> 905 self.variable[key] = value

File xarray\core\variable.py:866, in Variable.__setitem__(self, key, value)
--> 866 value = value.set_dims(dims).data

[Additional xarray stack trace continues...]

ValueError: dimensions ('channel',) must have the same length as the number of data dimensions, ndim=2
```

## Root Cause Analysis
The error occurs in the `_cal_complex_samples` method of `CalibrateEK80` at line 548, where echopype attempts to assign transmit duration values to GPT (General Purpose Transceiver) channels. The issue appears to be related to how dimensional information is preserved (or not preserved) when EK80 wide band data is saved to and loaded from netCDF format.

The specific line causing the failure is:
```python
tau_effective[ch_GPT] = beam["transmit_duration_nominal"][ch_GPT].isel(ping_time=0)
```

This suggests that the dimensional structure of the `transmit_duration_nominal` variable is not correctly maintained through the netCDF save/load cycle for EK80 wide band data.

## Test Data
The test file used to reproduce this bug is:
- **Filename**: `2107RL_CW-D20210813-T220732.raw`
- **Source**: NCEI (National Centers for Environmental Information)
- **Ship**: Reuben Lasker
- **Survey**: RL2107
- **Date**: August 13, 2021

## Reproduction Steps
You can reproduce this bug by running the provided `echopype-bug.ipynb` notebook, or by following these manual steps:

1. Load the raw EK80 file: `ed = ep.open_raw(raw_path, sonar_model="EK80")`
2. Convert to netCDF: `ed.to_netcdf(save_path=output_path)`
3. Reopen the netCDF file: `echodata = ep.open_converted(output_path)`
4. Attempt calibration: `ds_Sv = ep.calibrate.compute_Sv(echodata, waveform_mode="CW", encode_mode="complex")`
5. Observe the ValueError


## Environment
- **echopype version**: 0.10.1
- **Python version**: 3.11.3
- **xarray version**: 2025.7.0
- **Operating System**: Windows

