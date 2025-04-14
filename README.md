# sunet-cm-tooling-presentation

## Requirements:
* [presenterm](https://github.com/mfontanini/presenterm)
* [weasyprint](https://weasyprint.org) (For export to PDF)
* [The Sunet theme for presenterm](https://github.com/SUNET/presenterm-sunet-theme)

## Render
### Slides
> [!NOTE]
> With a white or very light terminal background for best compability with the translucent logo.

`presenterm slides/init.md`

### PDF
> [!NOTE]
> See [upstream docs](https://mfontanini.github.io/presenterm/features/pdf-export.html#pdf-page-size) regarding page size

`presenterm slides/init.md --theme sunet-export`
