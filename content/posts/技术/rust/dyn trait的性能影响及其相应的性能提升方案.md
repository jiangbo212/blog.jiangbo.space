---
title: "Dyn trait的性能影响及其相应的性能提升方案"
date: 2023-01-08T22:11:30+08:00
draft: false
categories: rust
keywords: rust "dyn trait" enum_dispatch "dynamic dispatch" 动态调度
---
本篇文章受meilisearch的issue [#2456](https://github.com/meilisearch/meilisearch/issues/2456) 启发。并主要参考以下两篇英文文章 
 + [a basic explanation of how dyn Traits work internally ](https://doc.rust-lang.org/std/keyword.dyn.html)
 + [a basice explanation of enum_dispath](https://docs.rs/enum_dispatch/latest/enum_dispatch) 。

Dyn trait一般用于多态操作，这块在面向对象编程中比较常见。我们想象一种场景，我们现在需要一个取值范围，这个范围既可以是线性的，即0, 1, 2, 3 ..... 1000; 也可以是对数范围。那么我们可以用一下代码来表示。

``` rust

    trait RangeControl {
        fn set_range(&mut self, value: f64);
        fn get_value(&self) -> f64;
    }

    struct LinerRange{
        position: f64;
    }

    struct LogarithmickRange {
        position: f64;
    }

    impl RangeControl for LinerRange {
        fn set_range(&mut self, value: f64) {
            self.position = value;
        }

        fn get_value(&self) -> f64 {
            self.position;
        }
    }

    impl RangeControl for LogarithmickRange {
        fn set_range(&mut self, value: f64) {
            self.position = value;
        }

        fn get_value(&self) -> f64 {
            (self.position + 1.).log2();
        }
    }

    fn main() {
        let v: Vec<Box<dyn RangeControl>> = vec![
            // linerRange or logarithmickRange
        ]

        // use v
    }

```

Dyn trait的性能损耗主要体现在动态调度(dynamic dispatch)方面。关于动态调度相关可参考[wiki-Dynamic_dispath](https://en.wikipedia.org/wiki/Dynamic_dispatch)。在rust的实现中表现为只有在运行时才能查找到真正需要执行的实现trait的方法, 那么在编译阶段就会因为编译器找不到真正执行的方法而无法进行优化。rust中的动态调度会额外的带来一个虚表的查询操作(vtable lookup)，它需要在执行的时候到虚表中去查找到真正执行的方法的指针，这会在运行时带来很高的损耗。

当然因为rust强大的枚举类型，上述的代码我们也可以用枚举来实现，这样编译器就可以正确的找到相应的类型，从而在编译阶段进行优化，这样运行时就不会带来性能损耗。

``` rust 
    enum RangeControlEnum {
        Liner(LinerRange),
        Logarithmick(LogarithmickRange),
    }

    impl RangeControl for RangeControl {

        fn set_position(&mut self, value: f64) {
            macth self {
                RangeControlEnum::Liner(inner_range) => inner_range.set_position(value),
                RangeControlEnum::Logarithmick(inner_range) => inner_range.set_position(value),
            }
        }

        fn get_value(&self) -> f64 {
            match self {
                RangeControlEnum::Liner(inner_range) => inner_range.get_value(),
                RangeControlEnum::Logarithmick(inner_range) => inner_range.get_value(),
            }
        }

    }

    fn main() {
        let v: Vec<RangeControlEnum> = vec![
            // Liner or Logarithmick
        ]

        // use v
    }
```

用枚举来表示虽然不会带来性能损耗，但是上述实现也会带来新的问题。如果我们继续为RangeControlEnum添加枚举，那么我们就需要为它的RangeControl实现的方法的模式匹配中添加新的选项。虽然这样做并不复杂，但在具体编写的时候会显的很琐碎。

同时我们也会发现在为枚举RangeControlEnum实现RangeControl时，所有的内部实现都显得很相似，这块其实并没有很好的进行封装。库(crate)enum_dispatch为我们代码很好的实现方案。使用enum_dispatch我们可以将枚举代码调整为以下的形式

``` rust
    #[enum_dispatch]
    trait RangeControl {
        // ....
    }


    // enum_dispatch会自动的实现std::convert::From从而为枚举里的每个对象自动创建名字，因此不需要我们手工添加名字
    #[enum_dispath(RangeControl)]
    enum RangeControlEnum {
        LinerRange,
        LogarithmickRange,
    }

```

因为enum_dispath的实现只是利用宏帮我们简化了枚举形式的实现。因此在性能上的表现是非常好的，相比于Dyn trait的实现，性能有高达10倍的提升。性能测试结果具体可参见enum_dispatch的crate页。

