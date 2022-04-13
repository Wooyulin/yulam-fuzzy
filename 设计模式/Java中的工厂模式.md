```java
package cn.yulam.factory;

/**
 * @author 5yl
 * date: 2022/4/13
 */
interface Game {
    void play();
}

class CF implements Game {
    @Override
    public void play() {
        System.out.println("play CF");
    }
}

class LOL implements Game {
    @Override
    public void play() {
        System.out.println("play LOL");
    }
}



/**
 * 简单工厂就是一个简单的方法，并不属于23种设计模式里面的一种
 * 优点：逻辑简单
 * 缺点：新建一个游戏，需要对工程进行修改，不符合开发原则
 */
class SimpleFactory {
    public Game build(String type) {
        switch (type) {
            case "CF":
                return new CF();
            case "LOL":
                return new LOL();
            default:
                return null;
        }
    }
}

// ==========================工厂方法模式 =================

/**
 * 工厂方法模式，满足迪米特法则、开闭原则、单一职责
 * 优点：新增游戏的时候，不会对原有代码进行修改
 * 缺点：类爆炸、每个调用者都要清晰知道自己调用的是哪个工厂
 */
interface GameFactory {
 Game build();
}

class CFFactory implements GameFactory {
    @Override
    public Game build() {
        return new CF();
    }
}

class LOLFactory implements GameFactory {
    @Override
    public Game build() {
        return new LOL();
    }
}


// ========================抽象工厂模式==========================

/**
 * 工厂管理多个产品类型的工厂如
 *   电子工厂：
 *      手机生产线工厂
 *      路由器生产线工厂
 *
 * 抽象工厂模式，和工厂方法模式的维度不一样，数据更上一级的管理
 * 工厂方法针对的是产品
 * 抽象工厂模式针对的是工厂
 * https://blog.csdn.net/qq_33732195/article/details/110101808
 */





public class Factory {

}
```