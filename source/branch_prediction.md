# 分支预测

## 基础知识

如果能在取指令阶段就可以预知本周期所取出的指令是否存在分支指令，并且可以知道他的方向以及目标地址的话，那么就可以在下个周期从分支指令的目标地址开始取指令，这样就不会对流水线产生影响，也避免作了无用功，提高了处理器的执行效率。这种不用等到分支指令的结果真的被计算出来，而是提前就预测结果的过程就是分支预测。

要进行分支预测，首先需要知道从I-Cache取出来的指令中，哪条指令是分支指令，这对于每周期取出多条指令的超标量处理器来说，更为不容易，需要从指令组中找出分支指令。

最容易想到的方法就是将指令组中的指令从I-Cache取出来之后，进行快速的解码，之所以称为快速，是因为只需要辨别指令是否是分支指令，然后将找到的分支指令对应的PC值送到分支预测器就可以对分支指令进行预测了。

但指令快速解码和分驻预测的过程都放在一个周期，严重影响了处理器的周期时间。

在流水线中，分支预测越靠前越好，如果指令从I-Cache取出来之后才进行分支预测，那么由于I-Cache中取出指令的过程可能需要多于一个周期才能够完成，当得到分支预测结果时，已经有很多后续的指令进入流水线，这样当预测失败时，这些指令都需要从流水线中抹掉(flush)，这样就降低了处理器的执行效率。因此分支预测的最好时机就是在当前周期得到取指令地址时，在取指令的同时进行分支预测，这样在下周期就可以根据预测的结果继续取指令。

### 两位饱和计数器(2-bit saturating counter)

基于两位饱和计数器的分支预测方法并不会马上使用分支指令上一次的结果，而是根据一条分支指令前两次的执行结果来预测本次的方向。这种方法可以用一个有着4个状态的状态机来表示，分别是
1. Strongly Taken
2. Weakly Taken
3. Weakly Not Taken
4. Strongly Not Taken

一般来讲，使用Strongly Not Taken或者Weakly Not Taken作为初始态。

可以使用格雷码对状态机进行编码，保证在状态转换时每次只有一位发生变化，这样可以减少出错的概率，并降低功耗。

### 分支预测失败时的恢复

## 动手实践