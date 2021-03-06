---
title: Azure Machine Learning 中的電腦視覺與分類模型準確度更高
description: 了解如何使用適用於電腦視覺的 Azure Machine Learning 套件，以改善電腦視覺影像分類、物件偵測和影像相似度模型的準確度。
services: machine-learning
ms.service: machine-learning
ms.component: core
ms.topic: conceptual
ms.reviewer: jmartens
ms.author: netahw
author: nhaiby
ms.date: 04/23/2018
ms.openlocfilehash: 6d6c540cd8711ae89784cdc797ca56d312522a83
ms.sourcegitcommit: 32d218f5bd74f1cd106f4248115985df631d0a8c
ms.translationtype: HT
ms.contentlocale: zh-TW
ms.lasthandoff: 09/24/2018
ms.locfileid: "46989306"
---
# <a name="improve-the-accuracy-of-computer-vision-models"></a>改善電腦視覺模型的準確度

[!INCLUDE [workbench-deprecated](../../../includes/aml-deprecating-preview-2017.md)] 

您可以使用**適用於電腦視覺的 Azure Machine Learning 套件**，以建置和部署影像分類、物件偵測和影像相似度模型。 深入了解此套件及其安裝方式。

在本文中，您會了解如何微調這些模型，以提高其準確度。 

## <a name="accuracy-of-image-classification-models"></a>影像分類模型的準確度

「電腦視覺套件」可以為各種不同的資料集提供非常好的結果。 不過，與大部分的機器學習專案一樣，若要為新資料集取得最好的結果，則需要仔細調整參數，以及評估不同的設計決策。 下列各節提供有關如何改善特定資料集準確度的指引，也就是最有希望先最佳化的參數、應該針對這些參數嘗試的值，以及要避免的陷阱。

一般而言，要為深入學習模型定型時，必須在定型時間與模型準確度之間取捨。 「電腦視覺套件」已經預先設定了預設參數 (請參閱下表中的第一列)。這些預設參數著重於快速定型，且通常能產生高準確度的模型。 準確度通常可以進一步改善，例如，使用較高的影像解析度或更深入的模型，但代價是定型時間會增加 10 倍以上。

建議您先使用預設參數、定型模型、檢查結果、視需要更正實況標註，之後才嘗試放慢定型時間的參數 (請參閱下表建議的參數值)。 雖然技術上不需要了解這些參數，但是仍建議您了解。


### <a name="best-practices-and-tips"></a>最佳做法和秘訣

* 資料品質：定型集和測試集都應具有高品質。 也就是說，已正確標註影像、移除了模稜兩可的影像 (例如，肉眼無法辨別顯示的影像是網球或檸檬)，而且屬性互斥 (亦即，每個影像只屬於一個屬性)。

* 在改進 DNN 之前，應該使用預先定型和固定的 DNN 作為特徵器 (Featurizer)，以便為 SVM 分類器定型。 「電腦視覺套件」支援此方法，而且定型不需要太長的時間，因為 DNN 本身不會遭到修改。 甚至是這種簡單的方法，通常都可達到良好的精確度，因此會有較強的基準。 接著，下一步就是改進應該可提高準確度的 DNN。

* 如果影像中的目標物件很小，則影像分類方法一般而言運作不佳。 在這種情況下，請考慮使用物件偵測方法，像是以 Tensorflow 為基礎的「電腦視覺套件」Faster R-CNN。

* 定型資料越多越好。 根據經驗法則，每個類別應該至少有 100 個範例，也就是「狗」有 100 個影像、「貓」有 100 個影像等等。您仍然可以使用較少的影像來定型模型，但可能不會產生良好的結果。

* 定型影像必須位於 GPU 所在的本機電腦上，而且必須位於 SSD 磁碟機 (而不是 HDD) 上。 否則，影像讀取延遲可能會大幅降低定型速度 (甚至達 100 倍)。


### <a name="parameters-to-optimize"></a>要最佳化的參數

尋找這些參數的最佳值非常重要，而且通常會大幅提升準確度：
* 學習速率 (`lr_per_mb`)：可以認為是正常運行的最重要參數。 如果 DNN 改進後定型集的準確度高於 5% 左右，最可能是學習速率太高，或定型循環週期 (epoch) 的數目太低。 特別是使用小型資料集時，DNN 往往會在定型資料上過適 (over-fit)，不過在實務上，這將會在測試集上產生良好的模型。 我們通常會使用 15 個循環週期，其中最初的學習速率會降低兩倍；定型時若使用多個循環週期，在某些情況下可以改善效能。

* 輸入解析度 (`image_dims`)：預設的影像解析度為 224 x 224 像素。 使用較高的影像解析度 (例如 500 x 500 像素或 1000 x 1000 像素) 通常會大幅提高準確度，但會使 DNN 改進變慢。 「電腦視覺套件」預期輸入解析度是 (色頻, 影像寬度, 影像高度) 的元組，例如 (3, 224, 224)，其中色頻的數目必須設為 3 (紅-綠-藍頻帶)。

* 模型架構 (`base_model_name`)：嘗試使用 ResNet-34 或 ResNet-50 之類更深入的 DNN，而非預設的 ResNet-18 模型。 Resnet 50 模型不僅更深入，而且其倒數第二層的輸出大小為 2048 浮點數 (相對於ResNet 18 和 ResNet 34 模型的 512 浮點數)。 讓 DNN 維持固定而非為 SVM 分類器定型時，增加維度可能會特別有幫助。

* 迷你批次大小 (`mb_size`)：迷你批次大小較高將會使得定型時間更快，但代價是增加了 DNN 記憶體耗用量。 因此，選取更深入的模型 (例如，ResNet-50 相較於 ResNet-18) 和/或更高的影像解析度 (500\*500 像素相較於 224\*224 像素) 時，通常必須減少迷你批次大小，以避免發生記憶體不足的錯誤。 變更迷你批次大小時，通常也需要調整如下表所示的學習速率。
* 中輟速率 (`dropout_rate`) 和 L2 正則項 (`l2_reg_weight`)：若要降低 DNN 過適，可以使用中輟速率 0.5 (「電腦視覺套件」中的預設值為 0.5) 或更高值，以及增加正則項權數 (「電腦視覺套件」中的預設值為 0.0005)。 但是請注意，特別是當使用小型資料集時，很難而且通常無法避免 DNN 過適。


### <a name="parameter-definitions"></a>參數定義

- **學習速率**：梯度下降學習期間使用的步進大小。 如果設定太低，模型將會採用許多循環週期來定型；如果設定太高，則模型將不會匯集成良好的解決方案。 通常會使用排程，以便在特定循環週期數目之後降低學習速率。 例如，學習速率排程 `[0.05]*7 + [0.005]*7 + [0.0005]` 同等於前 7 個循環週期使用初始學習率 0.05，後面接著另外 7 個循環週期使用降低 10 倍學習速率的 0.005，最後 1 個循環週期使用降低 100 倍學習速率的 0.0005 來微調模型。

- **迷你批次大小**：GPU 可以平行處理多個影像，以加速運算。 這些平行處理的影像也可以稱為迷你批次。 迷你批次大小越高，定型速度越快，但代價是增加 DNN 記憶體耗用量。

### <a name="recommended-parameter-values"></a>建議的參數值

下表提供的不同參數集，可針對各種影像分類工作，產生高準確度的模型。 最佳參數取決於特定的資料集以及使用的確切 GPU，因此下表僅供參考。 嘗試這些參數之後，也請一併考量超過 500 x 500 像素的影像解析度，或是 Resnet-101 或 Resnet-152 等更深入的模型。

表格中的第一列會對應至「電腦視覺套件」內設定的預設參數。 其他所有列則需要較長的時間才能定型 (如第一欄所示)，但優點是可提高準確度 (請見第二欄中三個內部資料集的平均準確度)。 例如，最後一列中的參數需要 5 到 15 倍的時間來定型，不過卻使得三個內部測試集的準確度從 82.6% 提升到 92.8% (平均)。

模型越深入且輸入解析度越高，則佔用的 DNN 記憶體越多，因此必須降低迷你批次的大小並增加模型複雜度，以避免發生記憶體不足的錯誤。 在下表中可以看出，每當減少兩倍的迷你批次大小時，將學習速率減少相同倍數將會很有幫助。 如果 GPU 的記憶體數量更少，則可能必須再減少一些迷你批次大小。

| 定型時間 (粗略估計) | 範例準確度 | 迷你批次大小 (*mb_size*) | 學習速率 (*lr_per_mb*) | 影像解析度 (*image_dims*) | DNN 架構 (*base_model_name*) |
|------------- |:-------------:|:-------------:|:-----:|:-----:|:---:|
| 1 倍 (參考) | 82.6% | 32 | [0.05]\*7  + [0.005]\*7  + [0.0005]  | (3, 224, 224) | ResNet18_ImageNet_CNTK |
| 2-5 倍    | 90.2% | 16 | [0.025]\*7 + [0.0025]\*7 + [0.00025] | (3, 500, 500) | ResNet18_ImageNet_CNTK |
| 2-5 倍    | 87.5% | 16 | [0.025]\*7 + [0.0025]\*7 + [0.00025] | (3, 224, 224) | ResNet50_ImageNet_CNTK |
| 5-15 倍        | 92.8% |  8 | [0.01]\*7  + [0.001]\*7  + [0.0001]  | (3, 500, 500) | ResNet50_ImageNet_CNTK |


## <a name="next-steps"></a>後續步驟

如需適用於電腦視覺的 Azure Machine Learning 套件的相關資訊：
+ 查看參考文件

+ 了解[適用於 Azure Machine Learning 的其他 Python 套件](reference-python-package-overview.md)