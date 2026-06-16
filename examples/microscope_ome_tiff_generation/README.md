# OME-TIFF Metadata Generation for Image Acquisition

*By Arjun Swani*

When using automated microscopy for large-area image stitching, high-resolution Z-stacking or multi-dimensional (multi-D) acquisition, managing the captured data can be a challenging task. A single automated run on a Zaber microscope can generate several individual image files along with separate metadata logs. Relying on separate image and metadata files can make the downstream analysis difficult and can increase the risk of critical data being lost. 

The [Open Microscopy Environment (OME) data model](https://ome-model.readthedocs.io/en/stable/index.html) was developed to solve this by structuring experiment data for biological microscopy imaging. While this model can be expressed as just metadata in [OME-XML](https://ome-model.readthedocs.io/en/stable/ome-xml/) files, it is most effective when embedded with image data to create [OME-TIFF](https://ome-model.readthedocs.io/en/stable/ome-tiff/specification.html) files.

This guide demonstrates how to use Python to combine an .ome.xml file with raw image data to create a compatible .ome.tiff file. This example was specifically developed to be used with the Zaber Launcher Microscopy app, which seamlessly exports image metadata as OME-XML files after image acquisition.

Using this Python code, you will be able to:
- Create and Modify OME-TIFF Datasets: Bind structured metadata (X/Y stage coordinates, Z-focus depth, exposure channels) directly to the corresponding image pixels using the widely adopted OME-TIFF specification to create compact, portable data sets. 
- Organize Complex Acquisitions: OME-TIFF organizes frames along TCZYX axes (Time, Channel, Z, Y, X), allowing massive multi-dimensional acquisitions and Z-stacks to be stored in a single, manageable file.
- Simplify Image Stitching & Analysis: Open the generated OME-TIFF files directly in standard bioimaging tools or libraries (like ImageJ/Fiji) for immediate processing, seamless image stitching, and advanced analysis.

This guide was developed specifically for use with the [Zaber Launcher Microscopy app](https://www.zaber.com/zaber-launcher) that exports image metadata as OME-XML files after image acquisition.

## Hardware Requirements

This example does not require any hardware to run. The example assumes that an image acquisition has been completed using a [Zaber microscope](https://www.zaber.com/products/microscopes).

## Dependencies / Software Requirements / Prerequisites

This example uses [`uv`](https://docs.astral.sh/uv/getting-started/installation/) to manage the virtual environment and dependencies. The dependencies are listed in `pyproject.toml`.

This example requires an image acquisition to be completed prior to running, preferably with Zaber Launcher Microscope App, and requires the following to run:

- OME XML metadata: Exported using the Zaber Launcher Microscope App
- Image Acquisition Directory: Contains acquisition images captured using third party imaging software. The example assumes all images have the same dimensions.

## Running the Script

- `-m, --ome-metadata`: Path to the `.ome.xml` metadata file.
- `-d, --acquisition-dir`: Directory containing acquisition images.
- `-o, --output-directory` (Optional): Output directory for generated `.ome.tiff` file.

To run the example:

```shell
cd examples/microscope_ome_tiff_generation/
uv run ome_tiff.py -m ./sample_data/xy.ome.xml -d ./sample_data -o ./output
```

## Customising Example

The `OMETiffWriter` class provided in the example can be modified to suit your specific use case. The example can be easily modified to extend supported image formats or add/customise metadata for the generated `.ome.tiff` file.

### Acquisition image Format / File Structure

`OMETiffWriter` assumes that images in the acquisition directory are named corresponding to the acquisition order for e.g `image001.png`, `image002.png` etc. If your files are sorted in a different way, you can override or modify the `get_acquisition_order` function to match your setup.

The example only looks for image file names matching the patterns in the `IMAGE_FORMAT_PATTERNS`. Modify this constant to filter or extend image files discovered.

### Modifying Metadata

`OMETiffWriter` uses the [`ome_types`](https://pypi.org/project/ome-types/) library to parse metadata XML into a Python object which is further modified in `modify_metadata`.

The code in `modify_metadata` looks at the image data to derive the `pixel_type`, `interleaved` and `samples_per_pixel` attributes for OME `Image` and `Pixels` objects. These are often required by OME-TIFF readers for rendering.

Additional logic to modify metadata can be added to this method.

```python
    def modify_metadata(self, sample: np.ndarray) -> str:

        # get OME metadata object
        ome = from_xml(self.metadata)
        pixel_type = self.DTYPE_TO_OME.get(sample.dtype) 

        if sample.ndim == self.NUM_CHANNELS_GREYSCALE:  
            # derive metadata from image shape
        else:  
            # ...

        for image in ome.images:
            # add derived metadata to metadata object
        
        # your metadata logic here
        
        return ome.to_xml()

```

## Notes

### TiffWriter

This example uses the [`tifffile`](https://github.com/cgohlke/tifffile) library to write the OME-TIFF file.

Even though OME-TIFF can support multi-file datasets or partial datasets, this example writes a single complete TIFF file. As specified in the OME TIFF documentation, the actual image data is stored in Tiff Image File Directories (IFDs, sometimes used interchangeably with pages). The `ImageDescription` of the first IFD contains the metadata.

The following snippet from `write_ome_tiff` writes the Tiff file:

```python
    with TiffWriter(ome_tiff_dir / self.metadata.with_suffix(".tiff").name, kind="generic") as tif:
        for index, frame in enumerate(self.get_acquisition_images()):
            if index == 0:
                metadata_str = self.modify_metadata(frame)
                tif.write(frame, contiguous=True, description=metadata_str.encode())
            else:
                tif.write(frame, contiguous=True)
```

While the `TiffWriter` class has native support for `OME` metadata, we do not use it in this example. Hence, the `kind` parameter of `TiffWriter` is `generic` not `ome`.
The `TiffWriter` `OmeXml` class is not comprehensive and does not support the full model. We instead use the `ome-types` library to parse the metadata file, perform operations on it and convert to a string that is written to the OME-TIFF file using `TiffWriter`.

In the `tif.write` function, the metadata is passed to the description parameter for the first image or IFD. The `contiguous` argument ensures that each image is written in the same series of the tiff file.

### Use of Custom Annotations

Zaber Launcher allows configurations of illuminators that are not supported by the OME-XML model. Zaber Launcher gets around this by putting the illuminator configuration in [Structured Annotations](https://www.openmicroscopy.org/Schemas/Documentation/Generated/OME-2016-06/ome_xsd.html#StructuredAnnotations) that allow additional information to be added to the OME metadata.

The [OME `Channel`](https://www.openmicroscopy.org/Schemas/Documentation/Generated/OME-2016-06/ome_xsd.html#Channel) assumes that each captured image plane will have a single wavelength light source which is likely in most cases. However, Zaber Microscopes additionally allow multi wavelength configurations by allowing multiple illuminators to be simultaneously on.

The current illuminator configuration is captured in structured annotations like so:

```xml
  <StructuredAnnotations>
    <MapAnnotation ID="Annotation:IlluminatorConfig:0" Namespace="Zaber">
      <Value>
        <M K="name">473A Green</M>
        <M K="hardwareChannelIndex">2</M>
        <M K="wavelengthNm">473</M>
        <M K="attenuationFraction">1</M>
      </Value>
    </MapAnnotation>
  </StructuredAnnotations>
```

Each different configuration has a corresponding annotation generated which is referenced under the OME `Channel` metadata tag for image planes. When single illuminator setups are used, the corresponding OME `Channel` metadata is filled in addition to the generated structured annotations.

While most third party readers will not be able to parse this information directly, it will be viewable under the OME metadata and can be parsed programmatically.

## Conclusion

Whether you are scanning wafers or performing deep-tissue Z-stacks, this Python implementation ensures that your Zaber microscope outputs immediately analyzable data. For more information about using the Zaber microscope, see the following resources: 
- [Zaber Launcher Microscope Tutorials](https://software.zaber.com/zaber-launcher/tutorials/Microscope)
- [Zaber Motion Library Microscopy API](https://software.zaber.com/motion-library/api/py/microscopy/)