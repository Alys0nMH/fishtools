# fishtools

[![Install and run](https://github.com/chaichontat/fishtools/actions/workflows/test.yml/badge.svg)](https://github.com/chaichontat/fishtools/actions/workflows/test.yml)

Tools for FISH analysis

## Installation

There are two environment files. One for

```sh
mamba env create -n fishtools -f environment_rsc_rapids_25.04.yml
mamba env update -n fishtools -f environment.yml
```

`mamba` is a drop-in replacement for `conda` that is faster and more reliable.
You can install `mamba` using `conda`:

```sh
conda install mamba -c conda-forge
```

## Compression

- `fishtools compress` is a command-line interface (CLI) tool that converts TIFF, JP2, and DAX image files to JPEG XL (JXL) files.
- `fishtools decompress` is a CLI tool that converts JXL files back to DAX files.

![Comparison of different compression quality](https://github.com/chaichontat/fishtools/assets/34997334/95230a08-4817-433d-a98d-67b5c442439d)

#### Usage

To use `fishtools`, simply run the `fishtools` command followed by the subcommand and path to the directory containing TIFF, JP2, or DAX files that you want to convert:

```sh
fishtools compress path/to/directory
```

By default, `fishtools compress` will convert all TIFF, JP2, and DAX files in the specified directory and its subdirectories. The converted JXL files will be saved in the same directory as the original files with the same name but with a `.jxl` extension.

You can also specify the quality level of the JXL files using the `--quality` or `-q` option. The quality level should be an integer between -inf and 100, where 100 is lossless. The default quality level is 99 (about 10x reduction in file size).

When the lossless option is selected, the output file is a `.tif` file with JPEG-XR encoding so that the file can be opened in ImageJ/BioFormats.

> BioFormats in ImageJ does not support JPEG-XL yet.
> It does support JPEG-XR which provides the same performance for lossless compression.
> JPEG-XR does not compress >8-channel images (compression scheme not in the specification).

If you want to delete the original files after conversion, you can use the `--delete` or `-d` option.

To use `fishtools decompress`, simply run the `fishtools decompress` command followed by the path to the JXL file or directory containing JXL files that you want to convert.
By default, `fishtools decompress` will convert all JXL files in the specified directory and its subdirectories.
The converted DAX files will be saved in the same directory as the original JXL files with the same name but with a `.dax` extension.

#### Examples

Convert all TIFF, JP2, and DAX files in a directory and its subdirectories to lossless TIFFs with JPEG-XR encoding:

```sh
fishtools compress path/to/directory
```

Convert all TIFF, JP2, and DAX files in a directory to JPEG-XL files and delete the original files:

```sh
fishtools compress path/to/directory --quality 99 --delete
```

Convert a single JXL file to DAX:

```sh
fishtools decompress path/to/file.jxl
```

Convert all JXL files in a directory to DAX:

```sh
fishtools decompress path/to/directory
```

## Probe ordering checklist

1. Verify simulation
2. BLAST some probes, make sure orientation is Plus/Minus.
3. Delete all old final files, both remote and local.
4. Run the script one last time.
5. Download said file and open to copy/paste into the Excel order sheet.
6. Save as a new file with today's date.
7. In the email, upload and redownload, verify that it's the same file.


# Image Preprocessing

This is instructions for image preprocessing to adata matrix for further analysis.

## Deconvolution

Prior to deconvolution, a basic directory needs to be copies to the working directory of the experiment. The basic file is applies a corrective field for each laser line and is specific for each microscope. Deconvolution command requires a GPU. This command outputs a subdirectory analysis/deconv/ which is the working directory for further preprocessing.

```sh
preprocess deconv batch . --basic-name=all
```

## Codebook Registration and Spot Calling

Registration and spot calling need to be done for each codebook. The config.json file can be created with empty brackets, but can have information for flags. Once a coodebook is used for registration, it is copied to a created codebook subdirectory.

```sh
preprocess deconv compute-range . --overwrite
preprocess register batch . --config=config.json --codebook=/path/to/codebook.json --threads=15
preprocess spots optimize . --codebook=/path/to/codebook.json --rounds=8 --threads=15
preprocess spots batch . --codebook=/path/to/codebook.json --threads=8 --split
preprocess stitch register . --idx=1 --max-proj --codebook=codebook --debug --overwrite
preprocess spots stitch . --codebook=/path/to/codebook.json --threads=15
```

## Non-bit Registration

Non-bit registration such as cell boundaries stains only require registration.

```sh
preprocess register batch . --codebook=/path/to/codebook.json --config=config.json --threads=2
preprocess stitch register . --idx=0 --max-proj --codebook=codebook 
```

## Checking Shifts and Thresholding

The check-shifts create an output subdirectory in analysis to verify tiels are registered correctly. There are a multiple outputs but primarily shift_layouts and L2 distance can be used to assess registration against the reference. Thresholding can used to assess distribution of spots. Each codebook needs to be ran separately. Thresholding is only for panel codebooks.

```sh
preprocess check-shifts . --codebook=codebooks/codebook.json
preprocess spots threshold . --codebook=codebooks/codebook.json
```

## Registration Troubleshooting

After check-shifts, there may be tiles missing or having major shifts from the reference (L2 distance greater than 10). Use check-shifts throughout correction to check tile improvement. The flags --threshold and --fwhm can be changed to improve registration. These values can range from 2-10 depending on the quantity and quality of fidicual spots. After registration correction, spot calling needs to run again with --overwrite flag for bit codebook panels. To check status and overwriting of command series, use preprocess status command.

```sh
# missing tiles
preprocess register run . tile --roi=roi --codebook=codebooks/codebook.json --debug --config=config.json --threshold=2 --fwhm=2
preprocess stitch register . roi --idx=1 --max-proj --codebook=codebook --debug --overwrite
preprocess check-shifts . --codebook=codebooks/codebook.json

# tile correction
preprocess register batch . --config=config.json --codebook=codebooks/codebook.json --overwrite --use-brightest=10 --offset-brightest=1 --only-median-gt=10 --fwhm=2 --threshold=2 --ref=2_10_18 --threads=2 --debug
preprocess check-shifts . --codebook=codebooks/codebook.json

# spot calling correction
preprocess spots batch . --codebook=codebooks/codebook.json --threads=8 --split --overwrite
preprocess stitch register . roi --idx=1 --max-proj --codebook=codebook --debug --overwrite
preprocess spots stitch . --codebook=codebooks/codebook.json --threads=8 --overwrite

# check progress
preprocess status . 
```

## Fuse Zarr

Use the non-bit (segmentation) codebook to create a zarr.

```sh
reprocess stitch fuse . --codebook=codebook/codebook.json --overwrite
```

## Segmentation

A cellpose model should already be trained and assessable. Edit the config.json file to add path of cellpose model with flags for running the command. Z index is for the center most z.

```sh
preprocess stitch n4 . --codebook=codebook --z-index=18
segment batch . --codebook=codebook
```

## Overlay Stains

Overlay function can be used to add intentsity of stains such as EdU fluorescence. Multiple panel codebooks can be used.

```sh
segment overlay all . --codebook=codebook1 --codebook=codebook2 --seg-codebook=reddot --intensity-codebook=edu --segmentation-name=output_segmentation.zarr
```

## Export Adata Matrix

```sh
segment export . --codebook=codebook1 --codebook=codebook2 --seg-codebook=reddot --segmentation-name=output_segmentation.zarr
```

## License

This project is licensed under the [MIT License](LICENSE).
