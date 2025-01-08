---
sidebar_position: 1
---

After struggling quite a bit to get this parallax effect working on both Chrome and Safari, on macOS and iOS, here’s the best solution I found. It’s more flexible than this method https://keithclark.co.uk/articles/pure-css-parallax-websites/demo3/ and uses modern CSS. 

![small_parallax.gif](./small_parallax.gif)

If you try it by modifying the background position of the image, for ex:
```css
@keyframes move-bg {
  from {background-position: 0% 50%;}
  to {background-position: 100% 50%;}
}
```
you get awful performance on safari :(

Here is the index.css file :
```css
.border {
    border: solid #5B6DCD 10px;
}

.parallaxBox {
    height: 90vh;
    /* we need to hide the image if go out the div. But if you set overflow: hidden
    it create an additionnal scroll context before the parent, so the animation scroll stop working
    "overflow: clip" do the same but without creating the scroll context */
    overflow: clip;
    /* There are child blocks with absolute positioning, but this absolute positioning needs to remain relative to the parent. Therefore, the parent must be given the   property: `position: absolute` or `relative`.  
If we set it to `absolute`, all the parallax frames end up at the very top, so we use `relative`. */
    position: relative;
}

.background-image {
    /* needs all 3 following props to let the image cover all the block */
    width: 100%;
    height: 100%;
    object-fit: cover;
    z-index: -1;
}

/* if we have : animation-timeline: scroll(); 
then all below images must be lifted up to compensate */
.offset {
    position: relative;
    top: -100%;
}

.small_height {
    height: 20vh;
}

.content {
    margin-top: 300px;
    font-size: 3vh;
    /* so the text is above the image */
    position: absolute;
    top:0px;
}

```


Here is the index.html file :

```html
<!DOCTYPE html>
<html>

	<head>
	    <title>Test</title>
	    <link rel="stylesheet" href="index.css">
	    <!-- add this polyfill if you want to make it work in safari -->
	    <script src="https://flackr.github.io/scroll-timeline/dist/scroll-timeline.js"></script>
	</head>

	<style>
	    /* the scroll-timeline polyfill only works if all animation related css are here and not in the css file */
	    @keyframes parallax {
	        to {
	            transform: translateY(100%); /* vh instead of % does not work with safari */
	        }
	    }
	
	    .parallax-anim {
	        animation: parallax linear;
	        animation-timeline: scroll();
          animation-fill-mode: forwards; /* prevent the last image the disapear on safari */
	    }
	</style>
	
	<body>
	    <section class="parallaxBox border">
	        <img src="assets/big-picture2.jpg" class="background-image parallax-anim">
	        <div class="content">section1</div>
	    </section>
	
	    <section class="border small_height">
	        <div>section2</div>
	    </section>
	
	    <section class="parallaxBox border">
	        <img src="assets/falaise.jpg" class="background-image parallax-anim offset">
	        <div class="content">section3</div>
	    </section>
	</body>
</html>
```

Ressources : 
https://www.youtube.com/watch?v=UmzFk68Bwdk
https://www.youtube.com/watch?v=Qj0Qx8HpNUo&t=5s
https://github.com/flackr/scroll-timeline?tab=readme-ov-file
https://www.joshwcomeau.com/animation/keyframe-animations/
https://www.w3schools.com/css/css3_animations.asp
https://openclassrooms.com/fr/courses/5919246-creez-des-animations-css-modernes/6340913-creez-des-animations-simples-avec-les-transitions
https://stackoverflow.com/questions/78096429/why-isnt-my-css-animation-timeline-view-working
