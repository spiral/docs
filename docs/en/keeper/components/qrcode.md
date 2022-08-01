# Forms: QR Code

UI component for QR Codes

They are located in toolkit bundle, that can be included like so:

```xhtml
<use:bundle path="toolkit:bundle"/>
```

This is also automatically included when using

```xhtml
<use:bundle path="keeper:bundle"/>
```

See [Usage Samples](https://github.com/spiral/app-keeper/blob/master/app/views/keeper/showcase/qrcode.dark.php) in demo
repository

## Usage

```xhtml
<ui:qrcode
        value="https://spiral.dev/"
        type="canvas"
        size="200"
        bgColor="#f8f9fa"
        fgColor="#49545f"
        ecLevel="H"
/>
```

`script` tag is optional, if specified, JSON inside will be used
for [TinyMCE options](https://www.tiny.cloud/docs/configure/)

| Parameter  | Required | Default | Description                                                                                                                |
|------------|----------|---------|----------------------------------------------------------------------------------------------------------------------------|
| value      | yes      | -       | Value to render                                                                                                            |
| type       | no       | svg     | Type of render, 'canvas' or 'svg'                                                                                          |
| size       | no       | 200     | Size of code                                                                                                               |
| bgColor    | no       | white   | Background color                                                                                                           |
| fgColor    | no       | black   | Foreground color                                                                                                           |
| ecLevel    | no       | M       | Error correction level                                                                                                     |
| logoUrl    | no       | -       | specifies URL of logo to render on top of code.                                                                            |
| logoHeight | no       | -       | Logo render height                                                                                                         |
| logoWidth  | no       | -       | Logo render width                                                                                                          |
| logoX      | no       | -       | position to render logo at                                                                                                 |
| logoY      | no       | -       | position to render logo at                                                                                                 |
| logoMargin | no       | -       | renders background color space around logo. Specify 0 to render only under logo. Omit to render logos with transparent BG. |

```xhtml
<ui:qrcode value="HK3ARG6MYFMIDDHB"/>
```

![QR Code](https://user-images.githubusercontent.com/16134699/103222713-b6914480-4935-11eb-8499-07e9b2d57b52.png)

```xhtml
<ui:qrcode
        value="https://spiral.dev/"
        type="canvas"
        size="300"
        bgColor="#f8f9fa"
        fgColor="#578fca"
        ecLevel="H"
        logoUrl="/logo.svg"
        logoHeight="50"
        logoWidth="40"
        logoX="130"
        logoY="125"
        logoMargin="0"
/>
```

![QR Code](https://user-images.githubusercontent.com/16134699/103222715-b729db00-4935-11eb-98f2-6c9adddd3124.png)

