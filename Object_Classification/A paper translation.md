#<center>亚马逊Pick挑战赛概率多类分割</center>
Rico Jonschkowski Clemens Eppner∗ Sebastian Hofer ¨ ∗ Roberto Mart´ ın-Mart´ ın∗ Oliver Brock
---
##摘要
&emsp;&emsp;我们提出了一种用于在真实的仓库中拣选操作中利用RGB-D数据进行多类分割的方法。
该方法计算每个像素的概率并把它们结合到一起来找到一个相干的对象分割。 即使当对象有半透明，反射，高度可变形，
具有模糊表面或由松散耦合的组件组成的特点时，它也可以在混乱的场景中可靠地分割对象。强大的性能源于对仓库操作本身问题结构的利用。 
作为我们赢得2015年亚马逊挑选挑战赛的一部分,我们提出的方法以此证明了它的能力。
我们提出了针对不同信息来源的贡献的详细实验分析，还将我们的方法与标准的分割技术进行比较，并评估可能的进一步增强算法的扩展性能。
我们开源了我们的软件和数据集。

##一、介绍
&emsp;&emsp;我们在本文中提出的多类分割方法是我们获得入围2015年亚马逊挑选挑战赛（APC）的关键组成部分[1]。
这种仓库物流挑战要求机器人通过自主地识别并从仓库货架的仓中拾取十二个物体来完成虚拟客户订单（参见图1）。
每个仓包含了从一组已知的25个对象中选出来的一到四个对象，为了定位并抓取目标物品，我们的系统在RGB-D图像上执行对象分割和分类（见图1c）。 
在比赛期间，我们的方法正确地分割和识别了所有十二个对象（图3），并使机器人能够成功抓取十个对象，超过其他全部25个队伍。

&emsp;&emsp;在比赛之前，我们对现成的用于目标识别，分割以及姿态估计的数据库的测试揭示了APC比赛操作中的实质性缺点。
这些发现在赛后对所有APC参赛队伍的调查中得到了证实：团队认为认知部分是APC的最困难的方面[2]。
这种困难与优秀的开源库（如PCL和OpenCV）的可用性恰好形成了相反的局面，并且遇到了一个看似简单的感知问题。
LINEMOD的一个标准目标检测和姿态估计的算法进行了更详细的测试，当应用于APC操作时，仅仅达到了32％的精度[3]。
即使为APC的操作进行专门的设计，它也只能达到60％的精度。

&emsp;&emsp;如果这些来自学术界和工业界的强大的研究团队无法在APC的环境中下利用计算机视觉几十年的研究的成就，我们就必须质疑我们的设定。
我们可以实现机器人的通用的,和任务无关的感知吗？在我们的APC解决方案中，我们偏离了以最通用形式解决感知问题的轨迹。
相反，我们只利用了手头上跟任务有关的知识组合成的一个简单的感知理论就成功了。
为了利用仓库操作的特性,我们的系统通过概率结合了许多简单的颜色和深度特征。

当然，由此产生的感知系统是为APC量身定制的特定任务型。
然而，我们提出了一个遵循一系列这样的增加通用性的任务特定系统的方法来应对鲁棒和更通用的机器人感知系统。
只有走这条路，我们才能确定通用机器人感知的关键部分。
为了实现这一目标，我们对我们的方法进行了广泛的事后评估，其中一些比较新奇的发现，例如,简单的图像处理可以就超越复杂的推理。
我们相信，这些发现不仅可以构建机器人的仓库场景，也为更强大的机器人感知系统提供宝贵的见解。
我们的贡献有三个方面：
> （i）我们描述我们APC比赛获胜的关键部分----对象分割系统;<br>
（ii）我们提出了超越比赛要求的彻底的实验评估;<br>
（iii）我们获得了关于如何建立特定任务的感知系统的更一般的教训

##二、相关工作

&emsp;&emsp;APC处感知问题是一般对象检测和分割问题的例子，是计算机视觉研究的热点。
诸如PASCAL VOC [4]或ImageNet [5]等主流视觉竞赛的结果表明，与我们的问题相关的解决方案目前正受到极大关注，
并且多年来获胜队伍一直都在稳步地减少错误率。
这些队伍中的一个共同的主题是使用可变形部分模型[6]或深层神经网络与大型数据集结合的滑动窗口方法[7]。
它们给出具有高度可能的对象位置的输出边界框，但也包含了一部分不是此目标的像素。
这使得这些表示难以在机器人这样精确性或至少保守来说形状估计对动作决策具有决定性的操作环境中使用。

&emsp;&emsp;多类分割通过识别图像中的每个像素属于哪个对象类来解决这个问题。
多类图像分割的流行方法是条件随机场（CRF，[8]）。
它们对局部（每像素或区域）信息编码,成对的统计偏好，并且定义其能量最小值来对应最可能的分割。
CRF提供了集成不同信息源（例如颜色，纹理，形状，位置和深度）的原则性方法，并且可以容易地和平滑约束[9,10]结合到一起。
与CRF类似，我们的方法以概率的方式组合不同的信息源。Sec. V-B显示了一个通用CRF和我们的方法之间的比较。

&emsp;&emsp;对象分割的更经典而有效的方法是直方图反投影[11]。
给定目标对象的颜色直方图，该方法通过用直方图的相应二进制计数替换每个像素颜色来把直方图反向投影到图像中。
具有高仓库计数的区域被假定为目标对象。在本文中，我们将直方图反投影扩展到概率版本，并纳入额外的非颜色特征。

&emsp;&emsp;在机器人操纵的流程中，对象检测的方法通常是为了估计完整的6D对象姿态，因此更多地依赖于深度数据。
检测和姿态估计可以基于CAD模型[12]，特征点直方图[13,14]，局部关键点描述符（如SIFT和SURF [15]）或边缘和法线信息来定位无纹理的对象，
如LINEMOD [16]. 这些方法都基于表顶部假设，但是当面对APC操作时出现的有限可见性和混乱情况,不能很好地收敛。
例如，LINEMOD [16]在每个仓格只有两个物品是[3]]显示已经出现的对应错误。虽然我们不估计对象的6D姿态，但我们的结果表明，
包含在分割中的形状信息已经让它足以被成功地抓住.

##三、问题分析

&emsp;&emsp;为了提出一个APC操作的多类分割方法，我们分析问题来试图找到有用的问题结构来解决识别挑战。

###A. APC操作中的挑战

> 没有对对象的单一辨别性属性：比赛选择了25个对象来代表反映仓库场景中存在的大多数常见品种。
但是没有单一的能用来识别的感知特征：一些对象具有不同的形状，但其他对象是可变形的;一些对象具有区别颜色，
但其他具有相似的颜色直方图或依赖视角变化;一些物体具有适于我们的RGB-D传感器的表面，但其他物体被包裹在塑料袋中。
我们通过结合各种特征来解决这个问题。

> 有限的对象可见性：基于相机的传感器只能从特定的摄像机位置获得架子上对象的局部视图。
周围的对象有时会遮挡一部分其他对象。我们通过分析对象不同姿势的传感器数据来解决这个问题。

> 不受控制的照明：视觉感知对光照条件十分敏感。在比赛现场，照明灯光直接从上方发出，相对于环境光来说非常明亮。
在这些条件下，图像在亮区几乎饱和，在剩余图像中显示黑色。
为了缓解这个问题，我们将RGB图像转换为HSV色彩空间，并包括了基于深度的特征。

> 部分3D测量：类似Kinect的传感器不能为反射或半透明材料提供可靠的3D测量，
例如塑料包装的物体或APC中使用的金属架。我们通过把缺失的深度值变成信息源作为分割的特征。

###B. APC设置中的有用的问题结构

> 少量的已知对象：由于完整的对象集合是可用的，并且事先已知，可以收集这些对象在不同方向和仓格位置的训练数据。

> 每个仓格的对象很少：由于仓格最多包含四个已知对象，因此我们可以在分割时忽略所有其他对象。
我们的方法自动使用对于存在于仓中的对象的特定子集来说最有判别性的特征。

> 已知搁架：由于物体放置在可由机器人跟踪的已知搁架中，因此我们可以使用与搁架相关的特征，
例如仓格中像素的高度或与跟踪搁架模型之间的距离。
这些特征有助于区分不同尺寸的物体,并且区分物体和货架。

##四、多类别分割理论

&emsp;&emsp;我们的多类别分割方法包括以下步骤：
使用各种特征，我们为每个像素和每个对象计算像素属于给定对象的概率。
然后我们使用高斯平滑结合邻近像素之间的概率，为每个像素分配最可能的对象标签，并为每个对象选择最可能的分割。
在最后一步中，我们使分割的大小与我们基于训练数据和顺序贪婪地分割对象的期望一致，消除已经分割的对象。
下面我们将详细解释所有步骤。

###A.基于第三部分的特征分析，我们通过结合区分对象的六个特征来描述每个像素：

> 颜色：基于色调饱和度（HSV）颜色表示的0 - 182范围内的离散值，其对于照明条件稳定性较好。
我们通过适当的阈值将HSV色彩空间投影到单个维度：对于具有区别颜色的像素（S> 90和V> 20），
我们将特征设置为H（范围从0-179），否则针对白色(> 200）为180，对于灰色（200> V> 50）为181，对于黑色像素（50> V）为182。

> 边缘：描述像素是否在视觉边缘附近的二进制特征。我们通过对图像应用Canny边缘检测并用小椭圆内核（直径为11像素）膨胀它来计算此特征。

> 丢失的3D信息：表示像素是否包含有效深度以及3D信息的二进制值。

> 到货架的距离：表示像素到货架上最近点的距离的连续值（以mm为单位）。
我们通过跟踪RGB-D图像中的货架来估计此值。为此，我们从基于机器人的定位和正向运动学的估计开始，
并使用迭代最近点（ICP）方法对其进行细化。忽略了没有有效深度信息的像素。

> 高度（3D）：表示像素到地面的最短距离的连续值（以mm为单位），类似于货架距离特征计算。没有有效深度信息的像素被再次忽略。

>　高度（2D）：描述投影到货架箱的开口前平面上的像素的高度的连续值（以mm为单位）。该特征近似3D高度信息，并且仅用于没有有效深度信息的像素。

###B.学习阶段

&emsp;&emsp;给定每个像素的6D特征向量，现在解释如何学习APC对象可能的特征。

> &emsp;&emsp;1）数据收集：在准备APC期间，官方提供了一个数据集，包括来自所有对象的多个视图的RGB-D图像以及对象的3D模型.
然而，我们发现，角度和照明条件使得它难以将模型从该数据集传输到我们的机器人。
此外，在这个数据集中，对象不在APC架内。因此，不能从该数据集中学习一些搁架相关特征的可能性以及搁架本身的模型。
因此，我们收集了一个和竞争情景非常类似的数据集。我们将对象放在不同的姿势货架上，收集RGB-D图像和估计的架子姿势。
最后，我们手动分割图像，直到我们对每个对象有足够数量的例子来覆盖他们可能的姿态（总共161个样本，平均每个对象6个样本）。

> &emsp;&emsp;2）计算特征似然：基于我们的数据集，我们为每个对象生成特征似然。
对于每个特征f（例如，颜色，高度），我们从属于手动标记的对象分割并归一化该直方图以获得关于可能值的似然性。
为了对特征值的微小变化具有鲁棒性，
我们使用了高斯核（平滑核的标准偏差：σcolor0-179= 3，σcolor180-182= 1，σdistto shelf = 7：5mm，σheight （3D）= 3mm，σ高度（2D）= 6mm）。
针对特征值的大范围变化的鲁棒性，我们将具有均匀分布的似然度混合，
其中我们对于不同的特征使用以下参数：（punicolor = 0：2，punidist to shelf = 0:05，puniedge = 0：4，punimiss3D = 0：2，puniheight（3D）= 0： （2D）= 0：8）。
因此，即使对于对象从未被观察到的特征值概率也是非零的，从而不完全排除某些对象。参数σf和punif定义了我们信任特征f的程度。

> &emsp;&emsp;3）像素标记和后处理：邻近的像素通常显示为相同的对象，因此应该分配给他们类似的目标概率。
为了合并这样的空间信息，我们用高斯核（σ= 4个像素）平滑每个对象的后图像。
此步骤与通过用圆盘[11]卷积其反向投影直方图和CRF中的成对电位来定位对象相关。
平滑步骤可以平衡比周围区域高得多或低得多的单个像素或小区域，这使得分割更加鲁棒。
在这里我们隐含地利用了APC对象紧凑并且占据了显著的图像区域的特性。

&emsp;&emsp;基于该平滑的后验图像，我们将每个像素i标记为属于具有最高后验的对象o，
并提取分配给同一对象的连接区域。在对象具有多个不连续的分割的情况下，我们选择包括该对象的平滑后图像中的值最大的那个。
作为后处理步骤，我们把分割变成凸包。此步骤包含缺失的对象部分，并反映大多数APC对象的凸度。
此外，我们查看片段的大小（像素数），并将其与数据集中此对象的片段大小进行比较。
如果分割被认为太大而不正确（大于其数据集中最大大小的1.2倍），我们会减少其后面的图像（减去0.05）并重新分配对象标签。
一直重复这一步骤，直到分割缩小到合理的大小。

&emsp;&emsp;最后的后处理步骤是基于以下思想的贪婪重新标记：如果我们对一个对象的分割有信心，
我们不必考虑该对象在图像的其余部分。我们通过顺序分割对象，从我们最确定的对象开始贪婪地开始（我们通过分割大小测量确定性）。
如果段大小与我们的数据集一致，我们假设我们已经找到了该对象具有高概率的正确分割，相应地减少其在分割外的后验（通过乘以0.2）并且重新归一化。
我们以相同的方式继续下一个最确定的对象，并继续，直到目标对象被处理