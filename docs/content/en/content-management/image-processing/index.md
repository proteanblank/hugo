---
title: Image processing
description: Resize, crop, rotate, filter, and convert images.
categories: []
keywords: []
---

## Image resources

To process an image you must access the file as a page resource, global resource, or remote resource.

### Page resource

{{% glossary-term "page resource" %}}

```text
content/
└── posts/
    └── post-1/           <-- page bundle
        ├── index.md
        └── sunset.jpg    <-- page resource
```

To access an image as a page resource:

```go-html-template
{{ $image := .Resources.Get "sunset.jpg" }}
```

### Global resource

{{% glossary-term "global resource" %}}

```text
assets/
└── images/
    └── sunset.jpg    <-- global resource
```

To access an image as a global resource:

```go-html-template
{{ $image := resources.Get "images/sunset.jpg" }}
```

### Remote resource

{{% glossary-term "remote resource" %}}

To access an image as a remote resource:

```go-html-template
{{ $image := resources.GetRemote "https://gohugo.io/img/hugo-logo.png" }}
```

## Image rendering

Once you have accessed an image as a resource, render it in your templates using the `Permalink`, `RelPermalink`, `Width`, and `Height` properties.

Example 1: Throws an error if the resource is not found.

```go-html-template
{{ $image := .Resources.GetMatch "sunset.jpg" }}
<img src="{{ $image.RelPermalink }}" width="{{ $image.Width }}" height="{{ $image.Height }}">
```

Example 2: Skips image rendering if the resource is not found.

```go-html-template
{{ $image := .Resources.GetMatch "sunset.jpg" }}
{{ with $image }}
  <img src="{{ .RelPermalink }}" width="{{ .Width }}" height="{{ .Height }}">
{{ end }}
```

Example 3: A more concise way to skip image rendering if the resource is not found.

```go-html-template
{{ with .Resources.GetMatch "sunset.jpg" }}
  <img src="{{ .RelPermalink }}" width="{{ .Width }}" height="{{ .Height }}">
{{ end }}
```

Example 4: Skips rendering if there's problem accessing a remote resource.

```go-html-template
{{ $url := "https://gohugo.io/img/hugo-logo.png" }}
{{ with try (resources.GetRemote $url) }}
  {{ with .Err }}
    {{ errorf "%s" . }}
  {{ else with .Value }}
    <img src="{{ .RelPermalink }}" width="{{ .Width }}" height="{{ .Height }}">
  {{ else }}
    {{ errorf "Unable to get remote resource %q" $url }}
  {{ end }}
{{ end }}
```

## Image processing methods

The `image` resource implements the  [`Process`],  [`Resize`], [`Fit`], [`Fill`], [`Crop`], [`Filter`], [`Colors`] and [`Exif`] methods.

> [!note]
> Metadata (EXIF, IPTC, XMP, etc.) is not preserved during image transformation. Use the `Exif` method with the _original_ image to extract EXIF metadata from JPEG, PNG, TIFF, and WebP images.

### Process

{{< new-in 0.119.0 />}}

> [!note]
> The `Process` method is also available as a filter, which is more effective if you need to apply multiple filters to an image. See [Process filter](/functions/images/process).

Process processes the image with the given specification. The specification can contain an optional action, one of `resize`, `crop`, `fit` or `fill`. This means that you can use this method instead of [`Resize`], [`Fit`], [`Fill`], or [`Crop`].

See [Options](#image-processing-options) for available options.

You can also use this method apply image processing that does not need any scaling, e.g. format conversions:

```go-html-template
{{/* Convert the image from JPG to PNG. */}}
{{ $png := $jpg.Process "png" }}
```

Some more examples:

```go-html-template
{{/* Rotate the image 90 degrees counter-clockwise. */}}
{{ $image := $image.Process "r90" }}

{{/* Scaling actions. */}}
{{ $image := $image.Process "resize 600x" }}
{{ $image := $image.Process "crop 600x400" }}
{{ $image := $image.Process "fit 600x400" }}
{{ $image := $image.Process "fill 600x400" }}
```

### Resize

Resize an image to the given width and/or height.

If you specify both width and height, the resulting image will be disproportionally scaled unless the original image has the same aspect ratio.

```go-html-template
{{/* Resize to a width of 600px and preserve aspect ratio */}}
{{ $image := $image.Resize "600x" }}

{{/* Resize to a height of 400px and preserve aspect ratio */}}
{{ $image := $image.Resize "x400" }}

{{/* Resize to a width of 600px and a height of 400px */}}
{{ $image := $image.Resize "600x400" }}
```

### Fit

Downscale an image to fit the given dimensions while maintaining aspect ratio. You must provide both width and height.

```go-html-template
{{ $image := $image.Fit "600x400" }}
```

### Fill

Crop and resize an image to match the given dimensions. You must provide both width and height. Use the [`anchor`] option to change the crop box anchor point.

```go-html-template
{{ $image := $image.Fill "600x400" }}
```

### Crop

Crop an image to match the given dimensions without resizing. You must provide both width and height. Use the [`anchor`] option to change the crop box anchor point.

```go-html-template
{{ $image := $image.Crop "600x400" }}
```

### Filter

Apply one or more [filters] to an image.

```go-html-template
{{ $image := $image.Filter (images.GaussianBlur 6) (images.Pixelate 8) }}
```

Write this in a more functional style using pipes. Hugo applies the filters in the order given.

```go-html-template
{{ $image := $image | images.Filter (images.GaussianBlur 6) (images.Pixelate 8) }}
```

Sometimes it can be useful to create the filter chain once and then reuse it.

```go-html-template
{{ $filters := slice  (images.GaussianBlur 6) (images.Pixelate 8) }}
{{ $image1 := $image1.Filter $filters }}
{{ $image2 := $image2.Filter $filters }}
```

### Colors

`.Colors` returns a slice of hex strings with the dominant colors in the image using a simple histogram method.

```go-html-template
{{ $colors := $image.Colors }}
```

This method is fast, but if you also scale down your images, it would be good for performance to extract the colors from the scaled down image.

### EXIF

Provides an [EXIF] object containing image metadata.

You may access EXIF data in JPEG, PNG, TIFF, and WebP images. To prevent errors when processing images without EXIF data, wrap the access in a [`with`] statement.

```go-html-template
{{ with $image.Exif }}
  Date: {{ .Date }}
  Lat/Long: {{ .Lat }}/{{ .Long }}
  Tags:
  {{ range $k, $v := .Tags }}
    TAG: {{ $k }}: {{ $v }}
  {{ end }}
{{ end }}
```

You may also access EXIF fields individually, using the [`lang.FormatNumber`] function to format the fields as needed.

```go-html-template
{{ with $image.Exif }}
  <ul>
    {{ with .Date }}<li>Date: {{ .Format "January 02, 2006" }}</li>{{ end }}
    {{ with .Tags.ApertureValue }}<li>Aperture: {{ lang.FormatNumber 2 . }}</li>{{ end }}
    {{ with .Tags.BrightnessValue }}<li>Brightness: {{ lang.FormatNumber 2 . }}</li>{{ end }}
    {{ with .Tags.ExposureTime }}<li>Exposure Time: {{ . }}</li>{{ end }}
    {{ with .Tags.FNumber }}<li>F Number: {{ . }}</li>{{ end }}
    {{ with .Tags.FocalLength }}<li>Focal Length: {{ . }}</li>{{ end }}
    {{ with .Tags.ISOSpeedRatings }}<li>ISO Speed Ratings: {{ . }}</li>{{ end }}
    {{ with .Tags.LensModel }}<li>Lens Model: {{ . }}</li>{{ end }}
  </ul>
{{ end }}
```

#### EXIF methods

Date
: (`time.Time`) Returns the image creation date/time. Format with the [`time.Format`]function.

Lat
: (`float64`) Returns the GPS latitude in degrees.

Long
: (`float64`) Returns the GPS longitude in degrees.

Tags
: (`exif.Tags`) Returns a collection of the available EXIF tags for this image. You may include or exclude specific tags from this collection in the [site configuration].

## Image processing options

The [`Resize`], [`Fit`], [`Fill`], and [`Crop`] methods accept a space-delimited, case-insensitive list of options. The order of the options within the list is irrelevant.

### Dimensions

With the [`Resize`] method you must specify width, height, or both. The [`Fit`], [`Fill`], and [`Crop`] methods require both width and height. All dimensions are in pixels.

```go-html-template
{{ $image := $image.Resize "600x" }}
{{ $image := $image.Resize "x400" }}
{{ $image := $image.Resize "600x400" }}
{{ $image := $image.Fit "600x400" }}
{{ $image := $image.Fill "600x400" }}
{{ $image := $image.Crop "600x400" }}
```

### Rotation

Rotates an image counter-clockwise by the given angle. Hugo performs rotation _before_ scaling. For example, if the original image is 600x400 and you wish to rotate the image 90 degrees counter-clockwise while scaling it by 50%:

```go-html-template
{{ $image = $image.Resize "200x r90" }}
```

In the example above, the width represents the desired width _after_ rotation.

To rotate an image without scaling, use the dimensions of the original image:

```go-html-template
{{ with .Resources.GetMatch "sunset.jpg" }}
  {{ with .Resize (printf "%dx%d r90" .Height .Width) }}
    <img src="{{ .RelPermalink }}" width="{{ .Width }}" height="{{ .Height }}">
  {{ end }}
{{ end }}
```

In the example above, on the second line, we have reversed width and height to reflect the desired dimensions _after_ rotation.

### Anchor

When using the [`Crop`] or [`Fill`] method, the _anchor_ determines the placement of the crop box. You may specify `TopLeft`, `Top`, `TopRight`, `Left`, `Center`, `Right`, `BottomLeft`, `Bottom`, `BottomRight`, or `Smart`.

The default value is `Smart`, which uses [Smartcrop] image analysis to determine the optimal placement of the crop box. You may override the default value in the [site configuration].

For example, if you have a 400x200 image with a bird in the upper left quadrant, you can create a 200x100 thumbnail containing the bird:

```go-html-template
{{ $image.Crop "200x100 TopLeft" }}
```

If you apply [rotation](#rotation) when using the [`Crop`] or [`Fill`] method, specify the anchor relative to the rotated image.

### Target format

By default, Hugo encodes the image in the source format. You may convert the image to another format by specifying `bmp`, `gif`, `jpeg`, `jpg`, `png`, `tif`, `tiff`, or `webp`.

```go-html-template
{{ $image.Resize "600x webp" }}
```

To convert an image without scaling, use the dimensions of the original image:

```go-html-template
{{ with .Resources.GetMatch "sunset.jpg" }}
  {{ with .Resize (printf "%dx%d webp" .Width .Height) }}
    <img src="{{ .RelPermalink }}" width="{{ .Width }}" height="{{ .Height }}">
  {{ end }}
{{ end }}
```

### Quality

Applicable to JPEG and WebP images, the `q` value determines the quality of the converted image. Higher values produce better quality images, while lower values produce smaller files. Set this value to a whole number between 1 and 100, inclusive.

The default value is 75. You may override the default value in the [site configuration].

```go-html-template
{{ $image.Resize "600x webp q50" }}
```

### Hint

Applicable to WebP images, this option corresponds to a set of predefined encoding parameters, and is equivalent to the `-preset` flag for the [`cwebp`] encoder.

Value|Example
:--|:--
`drawing`|Hand or line drawing with high-contrast details
`icon`|Small colorful image
`photo`|Outdoor photograph with natural lighting
`picture`|Indoor photograph such as a portrait
`text`|Image that is primarily text

The default value is `photo`. You may override the default value in the [site configuration].

```go-html-template
{{ $image.Resize "600x webp picture" }}
```

### Background color

When converting an image from a format that supports transparency (e.g., PNG) to a format that does _not_ support transparency (e.g., JPEG), you may specify the background color of the resulting image.

Use either a 3-digit or 6-digit hexadecimal color code (e.g., `#00f` or `#0000ff`).

The default value is `#ffffff` (white). You may override the default value in the [site configuration].

```go-html-template
{{ $image.Resize "600x jpg #b31280" }}
```

### Resampling filter

You may specify the resampling filter used when resizing an image. Commonly used resampling filters include:

Filter|Description
:--|:--
`Box`|Simple and fast averaging filter appropriate for downscaling
`Lanczos`|High-quality resampling filter for photographic images yielding sharp results
`CatmullRom`|Sharp cubic filter that is faster than the Lanczos filter while providing similar results
`MitchellNetravali`|Cubic filter that produces smoother results with less ringing artifacts than CatmullRom
`Linear`|Bilinear resampling filter, produces smooth output, faster than cubic filters
`NearestNeighbor`|Fastest resampling filter, no antialiasing

The default value is `Box`. You may override the default value in the [site configuration].

```go-html-template
{{ $image.Resize "600x400 Lanczos" }}
```

See [github.com/disintegration/imaging] for the complete list of resampling filters. If you wish to improve image quality at the expense of performance, you may wish to experiment with the alternative filters.

## Image processing examples

_The photo of the sunset used in the examples below is Copyright [Bjørn Erik Pedersen](https://bep.is) (Creative Commons Attribution-Share Alike 4.0 International license)_

{{< imgproc path="sunset.jpg" spec="resize 480x" alt="A sunset" />}}

{{< imgproc path="sunset.jpg" spec="fill 120x150 left" alt="A sunset" />}}

{{< imgproc path="sunset.jpg" spec="fill 120x150 right" alt="A sunset" />}}

{{< imgproc path="sunset.jpg" spec="fit 120x120" alt="A sunset" />}}

{{< imgproc path="sunset.jpg" spec="crop 240x240 center" alt="A sunset" />}}

{{< imgproc path="sunset.jpg" spec="resize 360x q10" alt="A sunset" />}}

## Configuration

See [configure imaging](/configuration/imaging).

## Smart cropping of images

By default, Hugo uses the [Smartcrop] library when cropping images with the `Crop` or `Fill` methods. You can set the anchor point manually, but in most cases the `Smart` option will make a good choice.

Examples using the sunset image from above:

{{< imgproc path="sunset.jpg" spec="fill 200x200 smart" alt="A sunset" />}}

{{< imgproc path="sunset.jpg" spec="crop 200x200 smart" alt="A sunset" />}}

## Image processing performance consideration

Hugo caches processed images in the `resources` directory. If you include this directory in source control, Hugo will not have to regenerate the images in a [CI/CD](g) workflow (e.g., GitHub Pages, GitLab Pages, Netlify, etc.). This results in faster builds.

If you change image processing methods or options, or if you rename or remove images, the `resources` directory will contain unused images. To remove the unused images, perform garbage collection with:

```sh
hugo --gc
```

[`anchor`]: /content-management/image-processing#anchor
[`Colors`]: #colors
[`Crop`]: #crop
[`cwebp`]: https://developers.google.com/speed/webp/docs/cwebp
[`Exif`]: #exif
[`Fill`]: #fill
[`Filter`]: #filter
[`Fit`]: #fit
[`lang.FormatNumber`]: /functions/lang/formatnumber/
[`Process`]: #process
[`Resize`]: #resize
[`time.Format`]: /functions/time/format/
[`with`]: /functions/go-template/with/
[EXIF]: https://en.wikipedia.org/wiki/Exif
[filters]: /functions/images/filter/#image-filters
[github.com/disintegration/imaging]: https://github.com/disintegration/imaging#image-resizing
[site configuration]: /configuration/imaging/
[Smartcrop]: https://github.com/muesli/smartcrop#smartcrop
