# Spiral Frontent Toolkit

## Forms

## Asset Management

## Embedded Layouts

## Simple Grids

```html
<spiral:grid source="<?= $uploads ?>" as="upload">
    <grid:cell value="<?= $upload->getId() ?>"/>
    <grid:cell value="<?= e($upload->getName()) ?>"/>
    
    <grid:cell><a href="#">Download</a></grid:cell>
</spiral:grid>
```

## Caching

```html
<spiral:cache lifetime=60>
  Some long code...
</spiral:cache>
```

## Miscellaneous
