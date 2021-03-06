[![StackOverflow](https://stackexchange.com/users/flair/7322082.png)](https://stackoverflow.com/users/5577765/rabbid76?tab=profile)

[![LinkedIn](image/LinkedIn-0077B5.png)](https://www.linkedin.com/in/gernot-steinegger/)

---

[GitHub - Rabbid76/PyGameExamplesAndAnswersRabbid76 - Rotate surface](https://github.com/Rabbid76/PyGameExamplesAndAnswers/blob/master/documentation/pygame/pygame_surface_rotate.md)  

[Stack Overflow- Rotating and scaling an image around a pivot, while scaling width and height separately in Pygame](https://stackoverflow.com/questions/70819750/rotating-and-scaling-an-image-around-a-pivot-while-scaling-width-and-height-sep/70820034#70820034)

---

# Rotate an image

Get the rectangle of the original image and set the position. Get the rectangle of the rotated image and set the center position through the center of the original rectangle. Return a tuple of the rotated image and the rectangle:

```py
def rot_center(image, angle, x, y):
    
    rotated_image = pygame.transform.rotate(image, angle)
    new_rect = rotated_image.get_rect(center = image.get_rect(center = (x, y)).center)

    return rotated_image, new_rect
```

Or write a function which rotates and `.blit` the image:

```py
def blitRotateCenter(surf, image, topleft, angle):

    rotated_image = pygame.transform.rotate(image, angle)
    new_rect = rotated_image.get_rect(center = image.get_rect(topleft = topleft).center)

    surf.blit(rotated_image, new_rect.topleft)
```

For the following examples and explanation I'll use a simple image generated by a rendered text:

```py
font = pygame.font.SysFont('Times New Roman', 50)
text = font.render('image', False, (255, 255, 0))
image = pygame.Surface((text.get_width()+1, text.get_height()+1))
pygame.draw.rect(image, (0, 0, 255), (1, 1, *text.get_size()))
image.blit(text, (1, 1))
```

An image ([`pygame.Surface`](https://www.pygame.org/docs/ref/surface.html)) can be rotated by [`pygame.transform.rotate`](https://www.pygame.org/docs/ref/transform.html#pygame.transform.rotate).

If that is done progressively in a loop, then the image gets distorted and rapidly increases:

```py
while not done:

    # [...]

    image = pygame.transform.rotate(image, 1)
    screen.blit(image, pos)
    pygame.display.flip()
```

![rotate 1](https://i.stack.imgur.com/AXgmY.gif)

This is cause, because the bounding rectangle of a rotated image is always greater than the bounding rectangle of the original image (except some rotations by multiples of 90 degrees).  
The image gets distort because of the multiply copies. Each rotation generates a small error (inaccuracy). The sum of the errors is growing and the images decays.

That can be fixed by keeping the original image and "blit" an image which was generated by a single rotation operation form the original image.

```py
angle = 0
while not done:

    # [...]

    rotated_image = pygame.transform.rotate(image, angle)
    angle += 1

    screen.blit(rotated_image, pos)
    pygame.display.flip()
```

![rotate 2](https://i.stack.imgur.com/vrhgt.gif)

Now the image seems to arbitrary change its position, because the size of the image changes by the rotation and origin is always the top left of the bounding rectangle of the image.

This can be compensated by comparing the [axis aligned bounding box](https://en.wikipedia.org/wiki/Minimum_bounding_box) of the image before the rotation and after the rotation.  
For the following math [`pygame.math.Vector2`](https://www.pygame.org/docs/ref/math.html) is used. Note in screen coordinates the y points down the screen, but the mathematical y axis points form the bottom to the top. This causes that the y axis has to be "flipped" during calculations  

Set up a list with the 4 corner points of the bounding box:

```py
w, h = image.get_size()
box = [pygame.math.Vector2(p) for p in [(0, 0), (w, 0), (w, -h), (0, -h)]]
```

Rotate the vectors to the corner points by [`pygame.math.Vector2.rotate`](https://www.pygame.org/docs/ref/math.html#pygame.math.Vector2.rotate):

```py
box_rotate = [p.rotate(angle) for p in box]
```

Get the minimum and the maximum of the rotated points:

```py
min_box = (min(box_rotate, key=lambda p: p[0])[0], min(box_rotate, key=lambda p: p[1])[1])
max_box = (max(box_rotate, key=lambda p: p[0])[0], max(box_rotate, key=lambda p: p[1])[1])
```

Calculate the "compensated" origin of the upper left point of the image by adding the minimum of the rotated box to the position. For the y coordinate `max_box[1]` is the minimum, because of the "flipping" along the y axis:

```py
origin = (pos[0] + min_box[0], pos[1] - max_box[1])

rotated_image = pygame.transform.rotate(image, angle)
screen.blit(rotated_image, origin)
```

![rotate 3](https://i.stack.imgur.com/IKZ6a.gif)

It is even possible to define a pivot on the original image. Compute the offset vector from the center of the image to the pivot and rotate the vector. A vector can be represented by [`pygame.math.Vector2`](https://www.pygame.org/docs/ref/math.html#pygame.math.Vector2) and can be rotated with [`pygame.math.Vector2.rotate`](https://www.pygame.org/docs/ref/math.html#pygame.math.Vector2.rotate). Notice that `pygame.math.Vector2.rotate` rotates in the opposite direction than `pygame.transform.rotate`. Therefore the angle has to be inverted:

Compute the offset vector from the center of the image to the pivot on the image:

```py
image_rect = image.get_rect(topleft = (origin[0] - pivot[0], origin[1]-pivot[1]))
    offset_center_to_pivot = pygame.math.Vector2(origin) - image_rect.center
```

Rotate the offset vector the same angle you want to rotate the image:

```py
rotated_offset = offset_center_to_pivot.rotate(-angle)
```

Calculate the new center point of the rotated image by subtracting the rotated offset vector from the pivot point in the world:

```py
rotated_image_center = (origin[0] - rotated_offset.x, origin[1] - rotated_offset.y)
```

Rotate the image and set the center point of the rectangle enclosing the rotated image. Finally blit the image :

```py
rotated_image = pygame.transform.rotate(image, angle)
rotated_image_rect = rotated_image.get_rect(center = rotated_image_center)

surf.blit(rotated_image, rotated_image_rect)
```

In the following example program, the function `blitRotate(surf, image, pos, originPos, angle)` does all the above steps and "blit" a rotated image to a surface.  

- `surf` is the target Surface

- `image` is the Surface which has to be rotated and `blit`

- `pos` is the position of the pivot on the target Surface `surf` (relative to the top left of `surf`)

- `originPos` is position of the pivot on the `image` Surface (relative to the top left of `image`)

- `angle` is the angle of rotation in degrees

This means, the 2nd argument (`pos`) of `blitRotate` is the position of the pivot point in the window and the 3rd argument (`originPos`) is the position of the pivot point on the rotating _Surface_:

![rotate 4](https://i.stack.imgur.com/qnDPP.gif)

![roatate and scale](https://i.stack.imgur.com/yLxBi.gif)

First the position of the pivot on the `Surface` has to be defined:

```py
image = pygame.image.load("boomerang64.png") # 64x64 surface
pivot = (48, 21)                             # position of the pivot on the image
```

When an image is rotated, then its size increase. We have to compare the [axis aligned bounding box](https://en.wikipedia.org/wiki/Minimum_bounding_box) of the image before the rotation and after the rotation.  
For the following math [`pygame.math.Vector2`](https://www.pygame.org/docs/ref/math.html) is used. Note in screen coordinates the y points down the screen, but the mathematical y axis points form the bottom to the top. This causes that the y axis has to be "flipped" during calculations  

Set up a list with the 4 corner points of the bounding box and rotate the vectors to the corner points by [`pygame.math.Vector2.rotate`](https://www.pygame.org/docs/ref/math.html#pygame.math.Vector2.rotate). Finally find the minimum of the rotated box. Since the y axis in pygame points downwards, this has to be compensated by finding the maximum of the rotated inverted height ("*max(rotate(-h))*"):

```py
w, h = image.get_size()
box = [pygame.math.Vector2(p) for p in [(0, 0), (w, 0), (w, -h), (0, -h)]]
box_rotate = [p.rotate(angle) for p in box]
min_x, min_y = min(box_rotate, key=lambda p: p[0])[0], max(box_rotate, key=lambda p: p[1])[1]
```

The computation of `min_x` and `min_y` can be improved by directly computing the x and y component of the rotated vectors by [trigonometric](https://en.wikipedia.org/wiki/Trigonometry) functions:

```py
w, h         = image.get_size()
sin_a, cos_a = math.sin(math.radians(angle)), math.cos(math.radians(angle))
min_x, min_y = min([0, sin_a*h, cos_a*w, sin_a*h + cos_a*w]), max([0, sin_a*w, -cos_a*h, sin_a*w - cos_a*h])
```

Calculate the "compensated" origin of the upper left point of the image by adding the minimum of the rotated box to the position in relation to the pivot on the image:

```py
origin = (pos[0] - originPos[0] + min_x - pivot_move[0], pos[1] - originPos[1] - min_y + pivot_move[1])
rotated_image = pygame.transform.rotate(image, angle)
screen.blit(rotated_image, origin)
```

In the following example program, the function `blitRotate(surf, image, pos, originPos, angle)` does all the above steps and [`blit`](https://www.pygame.org/docs/ref/surface.html#pygame.Surface.blit) a rotated image to the Surface which is associated to the display:  

- `surf` is the target Surface
- `image` is the Surface which has to be rotated and `blit`
- `pos` is the position of the pivot on the target Surface `surf` (relative to the top left of `surf`)
- `originPos` is position of the pivot on the `image` Surface (relative to the top left of `image`)
- `angle` is the angle of rotation in degrees

![rotate](https://i.stack.imgur.com/BmG1u.gif)

The same algorithm can be used for a [Sprite](https://www.pygame.org/docs/ref/sprite.html#pygame.sprite.Sprite), too.  
In that case the position (`self.pos`), pivot (`self.pivot`) and angle (`self.angle`) are instance attributes of the class.
In the `update` method the `self.rect` and `self.image` attributes are computed. e.g:

To rotate an image around a pivot point and zoom the image, all you have to do is scale the vector from the center of the image to the pivot point on the image by the zoom factor:

<s>`offset_center_to_pivot = pygame.math.Vector2(origin) - image_rect.center`</s>
```py
offset_center_to_pivot = (pygame.math.Vector2(origin) - image_rect.center) * scale
```

The final function that rotates an image around a pivot point, zooms and `blit`s the image might look like this:

```py
def blitRotateZoom(surf, original_image, origin, pivot, angle, scale):

    image_rect = original_image.get_rect(topleft = (origin[0] - pivot[0], origin[1]-pivot[1]))
    offset_center_to_pivot = (pygame.math.Vector2(origin) - image_rect.center) * scale
    rotated_offset = offset_center_to_pivot.rotate(-angle)
    rotated_image_center = (origin[0] - rotated_offset.x, origin[1] - rotated_offset.y)
    rotozoom_image = pygame.transform.rotozoom(original_image, angle, scale)
    rect = rotozoom_image.get_rect(center = rotated_image_center)

    surf.blit(rotozoom_image, rect)
```

The scaling factor can also be specified separately for the x and y axis:

```py
def blitRotateZoomXY(surf, original_image, origin, pivot, angle, scale_x, scale_y):

    image_rect = original_image.get_rect(topleft = (origin[0] - pivot[0], origin[1]-pivot[1]))
    offset_center_to_pivot = pygame.math.Vector2(origin) - image_rect.center
    offset_center_to_pivot.x *= scale_x
    offset_center_to_pivot.y *= scale_y
    rotated_offset = offset_center_to_pivot.rotate(-angle)
    rotated_image_center = (origin[0] - rotated_offset.x, origin[1] - rotated_offset.y)
    scaled_image = pygame.transform.smoothscale(original_image, (image_rect.width * scale_x, image_rect.height * scale_y))
    rotozoom_image = pygame.transform.rotate(scaled_image, angle)
    rect = rotozoom_image.get_rect(center = rotated_image_center)

    surf.blit(rotozoom_image, rect)
```

![Rotating and scaling an image around a pivot, while scaling width and height separately in Pygame](https://i.stack.imgur.com/KMZqD.gif)  

![Rotating and scaling an image around a pivot, while scaling width and height separately in Pygame](https://i.stack.imgur.com/QcX0r.gif)  
