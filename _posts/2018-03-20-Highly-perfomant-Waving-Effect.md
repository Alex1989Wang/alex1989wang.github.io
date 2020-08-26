---
layout: post
title: Design Waving Effect Of High Performance
date: 2018-03-20 08:47:23

tags:
- Animation 
- CALayer

category:
- English
---

<span style="border-bottom:2px dashed red;"><b>Even though the title of this post mentions performance, I could not find substantial evidence or data to back my speculation. So, I might update this post later on if extra related performance tests and measurements can be found.</b></span>

<!-- more -->

## Wave Animation

But, first of all, what is the waving effect mentioned in the title like?

<div align='center'>
<img 
src="/images/wave_effect_gif.gif" 
width="340" 
title = "waving effect in action"
alt = "waving effect in action"
align = center
/>
</div>

In fact, making such an effect programmatically is not that difficult. Most of, if not all, public repos which give you the ability to have such a waving effect from Github.com are using a timer (CADisplayLink or NSTimer) to drive the animation. 

Here are some of them:

- [BAFluidView](https://github.com/antiguab/BAFluidView)
- [YXWaveView](https://github.com/yourtion/YXWaveView)
- [WaterWave](https://github.com/LiweiDong/WaterWave)

Relying on such a timer to update the wave when there are just a few wave views might be ok. But, what if there are a whole bunch of them, like adding such a waving effect to each one of your table view cell? Will performance overhead occur? 

So, besides using a timer mechanism to drive and pace the animation, using `CAReplicatorLayer`, `CABasicAnimation` and `CAShapeLayer` might just do the trick. 

- Using replicator layer to hold two instances of wave-shape layer;
- Each instance of wave-shape layer has one or more complete waves (when using sin(x) function to draw the wave path, x must be one or multiple 2Pi); This is needed to make sure the second wave-shape layer instance's very start merges perfectly with the first's end;
- Using a `transform.translation.x` basic animation to drive the wave-shape layer instance;

Code snippets from [my demo project](https://github.com/Alex1989Wang/Demos/tree/master/DemoProjects/JWLayerTest/JWLayerTest/Tests/WaveEffect):

```objc
- (void)startWavingIfNeeded {
    CAReplicatorLayer *replicatorLayer = [self replicatorLayer];
    CGRect replicatorRect = replicatorLayer.bounds;
    if (CGRectIsEmpty(replicatorRect)) {
        return;
    }
    
    self.waveShapeLayer.frame = replicatorRect;
    CGFloat instanceWidth = CGRectGetWidth(replicatorRect);
    replicatorLayer.instanceTransform = CATransform3DMakeTranslation(instanceWidth, 0, 0);
    replicatorLayer.instanceDelay = 0;
    
    //setup shape
    [self setupWavePath];
    
    //restart animtion
    [self restartWaveShapeTranslation];
}

- (void)setupWavePath {
    CGRect sinPathCanvasRect = self.waveShapeLayer.bounds;
    sinPathCanvasRect = CGRectIntegral(sinPathCanvasRect);
    if (CGRectIsEmpty(sinPathCanvasRect)) {
        return;
    }
    
    [CATransaction begin];
    [CATransaction setDisableActions:YES];
    CGFloat canvasWidth = sinPathCanvasRect.size.width;
    CGFloat canvasHeight = sinPathCanvasRect.size.height;
    CGFloat canvasMidY = CGRectGetMidY(sinPathCanvasRect);
    UIBezierPath *sinPath = [UIBezierPath bezierPath];
    sinPath.lineWidth = 1.f;
    CGFloat yPos = 0;
    for (CGFloat xPos = 0.f; xPos <= canvasWidth; xPos += 1.f) {
        yPos = canvasMidY + sin((xPos)/canvasWidth * M_PI * 2) * canvasHeight / 8; //one-eighth
        if (fpclassify(xPos) == FP_ZERO) {
            [sinPath moveToPoint:(CGPoint){xPos, yPos}];
        }
        else {
            [sinPath addLineToPoint:(CGPoint){xPos, yPos}];
        }
    }
    
    //close path
    [sinPath addLineToPoint:(CGPoint){canvasWidth, canvasHeight}];
    [sinPath addLineToPoint:(CGPoint){0, canvasHeight}];
    [sinPath closePath];
    self.waveShapeLayer.path = sinPath.CGPath;
    [CATransaction commit];
}

- (void)restartWaveShapeTranslation {
    CAReplicatorLayer *replicatorLayer = [self replicatorLayer];
    CGRect replicatorRect = replicatorLayer.bounds;
    if (![self.waveShapeLayer animationForKey:kWaveShapeTranslationAnimationKey]) {
        //add translation animation to shape layer;
        CABasicAnimation *transAnim = [CABasicAnimation animation];
        transAnim.keyPath = @"transform.translation.x";
        transAnim.fromValue = [NSNumber numberWithFloat:0];
        transAnim.toValue = [NSNumber numberWithFloat:(-1.0 * replicatorRect.size.width)];
        transAnim.repeatCount = CGFLOAT_MAX;
        transAnim.duration = 1.0;
        transAnim.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionLinear];
        transAnim.removedOnCompletion = NO;
        transAnim.fillMode = kCAFillModeBoth;
        [self.waveShapeLayer addAnimation:transAnim forKey:kWaveShapeTranslationAnimationKey];
    }
    
    //if paused last time
    self.waveShapeLayer.speed = 1.0;
    self.waveShapeLayer.beginTime = 0;
    CFTimeInterval currentTime = CACurrentMediaTime();
    CFTimeInterval layerTimestamp = [self.waveShapeLayer convertTime:currentTime toLayer:nil];
    CFTimeInterval timeOffsetSincePaused = layerTimestamp - self.waveShapeLayer.timeOffset;
    self.waveShapeLayer.timeOffset = 0;
    self.waveShapeLayer.beginTime = timeOffsetSincePaused;
}

```

The wave path is drawn using the math sin(x) function; And, the parameters used in this function are chosen to serve the demo. If you want a different wave form, such as larger amplitude, you can tweak the parameters.  

<span style="border-bottom:2px dashed red;">The most important thing</span> is `sin((xPos)/canvasWidth * M_PI * 2)`. Here `(xPos)/canvasWidth * M_PI * 2` guarantees the wave will have just one complete cycle. So, when the `transform.translation.x` animation ends and repeats itself, the first shape-layer returns to itself original position, visually the wave never stops moving to left. 

## My Speculation

In [the Demo](https://github.com/Alex1989Wang/Demos/tree/master/DemoProjects/JWLayerTest/JWLayerTest/Tests/WaveEffect), `JWTimerWaveView` is used to mimic the behavior of a timer-driven wave animation. 

So, my speculation was replicator-layer-based animation will be somewhat better than timer-driven wave animation in terms of performance when used in a mass scale. So, my test was to put this two kinds of wave view into table view cells and scroll the table. 

However, when using `Core Animation Profiler`, the FPS value is almost the same for both of them. 

The only difference is `CPU Usage`, the timer-based animation uses about 7-10% of the cpu time constantly during the animation even when the table is not being scrolled. The replicator-layer-based animation only uses cpu time when scrolling the table, but when the table isn't being scrolled no cpu usage can be seen. 

<div align='center'>
<img 
src="/images/wave_effec_fps.png" 
width="220" 
title = "waving effect FPS"
alt = "waving effect FPS"
align = center
/>
</div>

<div align='center'>
<img 
src="/images/wave_effect_cpu_usage.png" 
width="600" 
title = "waving effect cpu usage"
alt = "waving effect cpu usage"
align = center
/>
</div>

The cpu usage can support my speculation in some way. But, I just feel the support isn't enough. If FPS drops largely when using time-driven animation, this might better prove my idea. But because timer-driven wave animation takes extra cpu time, when the App is doing something cpu-heavy, it's more likely to suffer a FPS drop than replicator-layer-based wave animation. 

This is open to discussion. More test ideas and opinions are welcome. 

## Reference

- [Demo](https://github.com/Alex1989Wang/Demos/tree/master/DemoProjects/JWLayerTest/JWLayerTest/Tests/WaveEffect)
- [BAFluidView](https://github.com/antiguab/BAFluidView)
- [YXWaveView](https://github.com/yourtion/YXWaveView)
- [WaterWave](https://github.com/LiweiDong/WaterWave)


