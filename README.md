# Template Dependency Inversion

Demonstrate template dependency extensibility problem and leading to a inversion
solution.

## Problem


```phtml
<div id="awesome-block">
    <?php echo $block->getChildHtml(); /** Extensions add here */ ?>
    <div id="awesome-child">
    </div>
</div>
```

```xml
<referenceBlock name="awesome-block">
    <block name="awesome-big-child" />
</referenceBlock>
```

`<block name="awesome-block" />` offers pre-designed extension:

- of specific area
- using a hook through `$block->getChildHtml()`

### Limitation of pre-designed hook

Hook is pre-designed. In the example, only the area that is in `awesome-block`
and before `awesome-child` can be extended.

When a 3rd-party module needs to to extend an area which had not been designed
to extend before. As a result, in this approach, the only way is to replace
the whole existing template.

A replacement to include a new hook for extension inside `awesome-block` and
after `awesome-child` is like below:

```phtml
<div id="awesome-block">
    <?php echo $block->getChildHtml('children'); ?>
    <div id="awesome-child">
    </div>
    <?php echo $block->getChildHtml('awesome-child-after'); ?>
</div>
```

When two 3rd-party module need to extend this area, given the original template,
they both need to replace the original template to introduce a new hook. At this
point, a conflict between these two module is created. Developer manual
intervention will be required to resolve the unnecessary conflict.

Core can offer more hooks as a limited workaround. But it is impossible to think
of all 3-rd party modules' needs unless a complete extensible template is
offered.

To offer a completely extensible template, it can go mad:

```phtml
<?php echo $block->getChildHtml('before'); ?>
<div id="awesome-block">
    <?php echo $block->getChildHtml('awesome-child-before'); ?>
    <div id="awesome-child">
        <?php echo $block->getChildHtml('awesome-child-children'); ?>
    </div>
    <?php echo $block->getChildHtml('awesome-child-after'); ?>
</div>
<?php echo $block->getChildHtml('after'); ?>
```

## Solution

Html itself is a layout. Css offers unambiguous selector to hook up with. In
additional to block based layout, offer html based layout as a hook-up point for
extension will highly reduce the conflicts among templates. The idea is like
AOP hooking up with an aspect call on specific class.

Using html as layout has an issue. Unlike block, html is deemed to change. When
changing block, it is easy to ring a bell and the developer will take into
account that backward compatibility may has broken and fix it. Html change will
not normally bring into attention unfortunately.

If we would like to use html as layout and css as selector, we must find a way
to tackle this issue.

A new layout is needed to workaround this issue.

`html-block` <- `block`

Instead of having a block to directly hook up with html layout, a `html-block`
layout is introduced. html-block uses css selector to create a hook on existing
template. A block that actually implement the extension will hook up with it.

The new layer provides a clear view of this fragile yet highly extensible layout.
On breaking html layout on upstream, extension only need to review and rebind
the broken html-block with new css selector.

html-block is going to be managed by extension rather than upstream. The same
selector may be used by different extensions and implemented as different
html-block. This is completely normal. It is not necessary to reuse html-block
on this level.

```xml
<htmlBlock name="around-awesome-child" selctor="#awesome-child"
    template="around-awesome-child.phtml" />

<referenceHtmlBlock name="around-awesome-child">
    <block name="awesome-big-child" />
</referenceHtmlBlock>
```

```phtml
<!-- around-awesome-child.phtml -->
<?php echo $block->getChildHtml('before'); ?>
<?php if ($show): ?>
    <?php echo $proceed(); ?>
<?php endif; ?>
<?php echo $block->getChildHtml('after'); ?>
```

Static analysis tool can be developed to verify html layout backward
compatibility breaking. A BC breaking happens when a known html-block's
selector is no longer valid in the new version of upstream. A warning should
be issued if in the new version that the tree of the selector has changed
however the node is still valid. This indicates that the semantic of the
selector may have changed therefore needing manual inspection.

It is necessary to compile html-block into the html layout. This will offer:

- Selector check to issue warning on node not found
- Speed up performance as xml processing is time consuming and should be done
before php processing.

Like AOP, a simple set of interceptions will be offered:

- beforeNode
- aroundNode
- innerNode
- afterNode

Selector of html-block is NOT supported to avoid recursion and late binding,
which may eat up performance.

Selector contain generated string is NOT supported. Or if we do, will need
double binding.

```phtml
<div id="<?php echo $prefix; ?>-awesome-<?php echo $block->getSuffix(); ?>" />
```

The only way to resolve `$prefix` or `getSuffix()` is to render the template
once. This means, to support this feature, we will need to do
render-bind-render. As the variable is deemed to be change on runtime, we
cannot pre-process the render-bind-render. Bringing this overhead to runtime
is not acceptable and thus giving up this feature.

Dynamically created block is also not supported because it requires a
render-bind-render to resolve too.
