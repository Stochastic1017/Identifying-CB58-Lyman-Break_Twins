# Finding the Cosmic Needle: Unraveling CB58 Resemblance in a Galactic Haystack

## Introduction

This project involves the identification of a novel gravitationally lensed Lyman-break galaxy that closely mirrors the target spectrum CB58. Lyman-break galaxies are those undergoing active star formation at high redshifts, selected based on the distinct appearance of the galaxy in various imaging filters, influenced by the position of the Lyman limit. The spectral data were captured through the Sloan Digital Sky Survey (SDSS), a significant multi-spectral imaging and spectroscopic redshift survey conducted with a dedicated 2.5-m wide-angle optical telescope at the Apache Point Observatory in New Mexico, United States. The dataset was generously provided by Christy Tremonti, an astronomer affiliated with the University of Wisconsin - Madison and a member of the Sloan Digital Sky Survey III (SDSS-III) collaboration. There are approximately 2.5 million spectral data (in the form of `.fits` files $\approx$ 281 gigabytes), and the top ten closest spectra were found by calculating red-shifted distance metrics for each spectral data in parallel using University of Wisconsin - Madison's Center For High-Throughput Computing via HTCondor. 

## Reading and interpreting .fits files in R

The R library `FITSio` contains the function `ReadFrameFromFITS()` that allows us to load .fits as a dataframe in R. Supposing we wish to load an `arbitrary.fits` as a dataframe, we write the following code:

``` r
require("FITSio")
df <- readFrameFromFITS("arbitrary.fits")
print(df)
```

The data frame for each of the spectra has these following columns:

1. `flux` is light intensity at a given wavelength. It is theoretically nonnegative, but with noise it can be negative.
2. `loglam `is `log(x=wavelength, base=10)`, so `wavelength = 10^loglam`. 
3. `ivar` (“inverse variance”) is $\frac{1}{S_i^2}$ where $S_i^2$ is an estimated variance of the $i$-th flux.
4. `and_mask` is 0 for a good observation. We exclude data with nonzero `and_mask`.
5. `or_mask` is 0 for a good observation.
6. `wdisp` is the resolution of the spectrograph at that wavelength.
7. `sky` is the spectrum of the sky, mostly already subtracted out from flux.
8. `model` is SDSS’s hypothesis of the true spectrum, shifted to the redshift of the object.

We use only `flux` and its corresponding `index` for this project. 

## Standardization

We standardize all `flux` values as follows before computing a distance metric:
$$z = \frac{x - \mu}{\sigma} \sim N(0,1)$$

``` r
standardize <- function(flux)
{
  return (scale(flux, center = mean(flux), scale = sd(flux)))
}
```

## Distance Metric

We implement Minkowski distances with p=2, i.e., euclidean distances at each flux of a spectra red-shifted.

``` r
minkowski  <- function(x, y, p)
{
  return (sum(abs((x-y)^p))^(1/p))
}
```

Our target spectra (cB58) has 2181 size vector values of `flux`. For noisy spectras that we compare to cB58, if the vector size is smaller than 2181, it is ignored. Else we compute distances at each red-shifted cB58 onto that noisy spectrum until we find the minimum distance and save the result.

``` r
standardize_minkowski <- function(cB58, spectra, p)
{
  n <- length(cB58)
  m <- length(spectra)
  dist  <-  c()

  if (n > m) # cB58 is larger than the other spectra
  {
    next
  }
  
  else (m > n)
  {
    cB58 <- standardize(cB58)
    for (i in 1:(m-n+1)) # Red-shifting at each index
    {
      temp  <- minkowski(cB58, standardize(spectra[i:(i+n-1)])[, 1], p)
      dist  <- append(dist, temp)
    }
  }
  return (c(min(dist), which(dist == min(dist))))
}
```
To allow the code to loop through each .fits file in a directory, and write an output .csv file whose name is the data directory name in the following format:
* **distance**: your measure of the distance from this spectrum to the template.
* **i**: the index in the spectrum at which your alignment with the template begins (red-shifted units).
* **spectrumID**: the spectrum ID, e.g., spec-1353-53083-0579.fits

``` r
files <- list.files(dir, pattern = 'fit*') # save all files as a list

for (file in files)
{
  cat("On File:", file, "\n")
  path_to_file = paste(sep = "", sprintf('%s/', dir), file)
  noisy = readFrameFromFITS(path_to_file) # Interested spectrum in this iteration
  result  <- rbind(result, c(standardize_minkowski(cB58$FLUX, noisy$flux, p = 2), file))
}

result <- na.omit(result)
write.csv(result, file = sprintf("%s.csv", dir), row.names = FALSE)
```

## Visualizing Template cB58 Spectrum

The standardized template cB58 spectra looks as follows:

<img src="https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/images/Standardized_cB58.png" width="700" height="500">

## Data Preprocessing

The 2.5 million .fits files were written into 2500 .tgz files, where each .tar file contained approximately 2459 .fits files ($\approx$ 100 mb each), and the data was stored in one of CHTC's approved distinct location (located in `~/data/tgz`). The template spectra is named as `cB58_Lyman_break.fit` (located in `~/data`), and each of the .tar file is of the form `[0-9][0-9][0-9][0-9].tgz` (example 5377.tgz).

## Bash and Shell Scripts

Various bash/shell scripts were written that performed various tasks before, during, and after running parallel jobs to find the top 10 closest spectras to cB58:

1. [`list.sh`](https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/shell/list.sh): Finds out the names of all .tgz files in the directory `~/data/tgz`, and writes them in the directory `~/minkowski/files`. This is done BEFORE job is submitted via HTCondor.
2. [`executable.sh`](https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/shell/executable.sh): Unpacks `R` (4.1.3), unpacks the `FITSio` package, tells bash where `R` and its packages are, unpacks the current .tgz file (like 3586.tgz), and runs [`minkowski_spectra.R`](https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/R/minkowski_spectra.R) on that directory (like 3586). This is done DURING the job, and at each compute node of the CHTC.
3. [`merge.sh`](https://github.com/Stochastic1017/Identifying-CB58-Lyman-Break-Twins/blob/main/shell/merge.sh): merges all 2459 .csv files into one, and writes the best 100 spectra to `100_minkowski_best.csv`. This is done AFTER all the parallel jobs are run.
4. [`pull_fits.sh`]():
