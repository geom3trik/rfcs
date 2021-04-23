# Feature Name: ui-layout

## Summary

The aim of this RFC is to provide an overview of the concepts and challenges in building a flexible layout system for bevy, as well as to provide a technical description of a proposed algorithm. 

## Motivation

UI layout involves the positioning and sizing of UI 'elements' within the application window. In a declarative UI framework, made popular by modern web frameworks, the user declares what elements should be placed on the screen, with a minimal set of layout properties, and a layout algorithm or system arranges those elements into the final visual output. Ideally, layout is done in such a way that the user can declare the size and position of elements in a flexible way so as to accomodate different screen sizes. This is esspecially important in the context of a multi-platform application such as a game, but also for an editor which can bve viewed at different resolutions and which could utilise the same layout algorithm.

At the heart of a UI layout algorithm is the concept of a visual tree or hierarchy of elements. The document object model (DOM) from the web world is an example of such a tree. This concept allows for parenting of UI elements so that the size and position of the parent could affect in some way the sizes and position its children. This concept is what allows for a 'flexible' layout system which is able to adapt to different output sceen sizes.

## Guide-level explanation

### Overview
An overview of the layout system for working with stacks, in which child elements are positioned one after another.

1. For stacking, a parent element arranges its children into a vertical stack (`layout-type: column`) or a horizontal stack (`layout-type: row`).

![layout_type](https://geom3trik.github.io/tuix-book/images/layout_type3.png)

2. A child element can be positioned by adding space to the `left`, `right`, `top`, or `bottom` of an element.

![spacing](https://geom3trik.github.io/tuix-book/images/spacing3.png)

3. A child element is positioned relative to its stack position (`position-type: parent-directed`), or relative to the top-left of its parent (`position-type: self-directed`).

![position_type](https://geom3trik.github.io/tuix-book/images/position_type.png)

4. Spacing can be expressed in pixels, percentage, stretch, or auto (see 8). Stretch causes the spacing to be flexible, occupying a proportion of the remaining free space. By default, spacing is set to auto.

![spacing_units](https://geom3trik.github.io/tuix-book/images/position3.png)

5. The size of an element is determined by its `width` and `height`, which can also be expressed in pixels, percentage, stretch and auto. By default, size is set to stretch.

![size_units](https://geom3trik.github.io/tuix-book/images/size3.png)

6. When size is set to auto, an element will 'hug' its children in that axis. This is also true for `min-width`/`min-height` (see 10).

![auto_size](https://geom3trik.github.io/tuix-book/images/auto_size.png)

7. A parent can add space around its stacked children with `child-space`, or specified individually with `child-left`, `child-right`, `child-top`, `child-bottom`, or add space between its child elements with `child-between`.

![child_spacing](https://geom3trik.github.io/tuix-book/images/child_spacing.png)

8. Child spacing applies to elements with spacing set to auto.

![auto_spacing](https://geom3trik.github.io/tuix-book/images/auto_spacing.png)

9. Spacing set in pixels, percentage, or stretch overrides child spacing.

![spacing_override](https://geom3trik.github.io/tuix-book/images/spacing_override.png)

10. All spacing and size properties have corresponding minimum and maximum constraint properties. For example, for `left` there is `min-left` and `max-left`. By default spacing and size minimums and maximums are set to auto. This will cause elements to `hug` their children if the size if flexible and goes below the minimum.

// TODO - Grids

## Reference-level explanation

### Terminology

#### Coordinate System
For the explanation to follow, we assume a cartesian coordinate system with an x axis pointing from left to right and a y axis pointing from top to bottom, with the origin in the top-left corner of the screen or parent element. This choice is somewhat arbitarary, taking inspiration from other UI frameworks, but ultimately can be changed before the layout algorithm is finalized.

#### Bounding Box
The proposed layout algorithm assumes that elements are contained within an axis-aligned bounding box. The purpose of the layout algorithm then is to calculate the size and position of the element bounding boxes based on a finite set of user-defined layout properties on each of the elements.

### Axes
A concious decision has been taken to define properties which act in either the horizonal or vertical axes. This is in contrast to systems like flex-box which define properties along a 'main' and 'cross' direction which change depending on the 'flex-direction'. Consequently, this proposed algorithm sacrifices easy transposability for clarity and ease-of-use.

### Layout Type and Position Type
By default, elements within a parent are arranged into a stack. A stack is a collection of elements placed one after another in either a horizontal (row) or vertical (column) direction. The direction of the stack is determined by the `layout-type` property, which can take on the value of `row`, `column`, or `grid`. A grid layout is separate to a stack and will be discussed later.

The `position-type` property determines what point the movement of an element is relative to. By default this property is set to `parent-directed`, and positioning an element results in shifting said element relative to its normal stack position. Alternatively, `self-directed` 'removes' the element from the stack and allows said element to be positioned relative to the top-left corner of the parent element. Other elements in the stack are layed out as if the `self-directed` elements were not in the stack.

### Positioning (Spacing)
Positioning elements involves adding space to each side of an element relative to the edges of the containing parent. For example, to position an element at x = 200 px and y = 100 px from the top-left corner of its parent, the user would add 200 pixels of space to the `left` side of the element and 100 pixels of space to the `top` side of the element. In this way positioning is synonymous with spacing around an element, and thus can be used to replicate margins from web layout. There are four spacing properties, `left`, `right`, `top` and `bottom` and an option to set all the properties should be provided.

Spacing is described with a `Units` enum which has four possible variants:
1. Pixels(f32)
2. Percentage(f32)
3. Stretch(f32)
4. Auto

The `Pixels` variant allows for spacing to be specified in screen space coordinates. The value is a floating point number, as opposed to an integer, to allow for smooth animations of spacing. What is not clear at this stage is how DPI factor, or scale factor, of the users display should be accounted for when specifying spacing and sizes in pixels. An alternative unit should probably be used which takes this into account.

The `Percentage` variant allows allows for spacing to be specified as a fraction of the parent size in the same axis. For example, a percentage spacing applied to the `left` or `right` properties will result in a spacing which is a fraction of the parent `width`. Likewise, a percentage `top` or `bottom` spacing will be a fraction of the parent `height`. This is true for both `parent-directed` and `self-directed` child elements.

The `Stretch` variant allows for spacing to be specified as a proportion of the 'free-space' available in that axis. Free-space for a particular axis is calculated as the parent size minus the 'used-space', where used-space is the sum of size and spacing which is described by pixels or percentages. For example, a parent with a width of 400 pixels, with two children each with widths of 100 pixels, and assuming 0 spacing, would have a free-space of: 400 - 100 + 100 = 200 pixels. The proportion of the free-space used by an element set to `Stretch` is the stretch value divided by the sum of all stretch values in the same axis. Thus final_size = (stretch / stretch_sum) * free_space. The various uses of the `Stretch` variant will be explored in more detail later on.

The `Auto` variant allows for spacing to be overriden by the parent. This will be explored in the Child Spacing section later on.

### Sizing
As well as positioning the elements, the layout algorithm is also responsible for assigning them a size. For this only two properties are required `width` and `height`. As with spacing the width and height properties are assigned a `Units` variant. The variants work in the same way as described previously with the exception of the `Auto` variant. For sizing, the `Auto` variant causes the element to 'hug' its child elements. The final width and height of the element is thus dependent on its stack direction. For a vertical stack the final height of the element when set to `Auto` will be the sum of the heights of its child elements, while the width will be the maximum of the widths of its child elements. The inverse is true for a horizontal stack, with the final width being the sum of the widths of the child elements and the final height being the maximum of the child element heights. The ability for an element to 'hug' its children is important for container widgets that must grow with the child elements added to it. 

### Flexible Layouts
With the spacing and sizing behaviour described previously, it is possible to build a UI layout which is flexible by specifying a non-zero number of space and size properties to be `Stretch`. 

// TODO - More here


### Child Spacing
For a parent with multiple child elements arranged in a stack it becomes neccessary to apply spacing around all of the child elements as well as between them in an easy way. While it is possible to acheive this by applying soacing to each of the individual elements, the proposed layout algorithm includes the concept of `child-space` which is an override for child elements with an individual spacing set to `Auto`. There are six child spacing proeprties: `child-left`, `child-right`, `child-top`, `child-bottom`, `row-between`, `column-between`. The first four are synonomous with the individual spacing counterparts. For example, `child-left` will override the spacing of all child elements with a `left` spacing set to `Auto`.

The two different properties are `row-between` and `column-between`. These properties also override individual child spacing but only on elements that are not on the 'outside' of a stack. For a non-wrapping stack these would be the firt and last child elements. For example, for a horizontal stack, `row-between` overrides both the `left` and `right` properties of the child elements to apply spacing between them. As with individual spacing the child spacing properties also use `Units`, where each variant behaves in the same was as described for individual spacing, apart from the `Auto` variant which has no effect.

### Wrapping

// TODO - Wrapping here

### Comparison with Flexbox
Currently, bevy UI relies on flexbox to perform layout of UI entities. Therefore, it is worth spending some time comparing flexbox to the layout algorithm proposed in this RFC. 

// TODO - comparison here

### Grids
// TODO

## Drawbacks

// TODO
Why should we *not* do this?

## Rationale and alternatives
// TODO

## \[Optional\] Prior art

// TODO

## Unresolved questions

// TODO

## \[Optional\] Future possibilities

// TODO
