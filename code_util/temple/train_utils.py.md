## 关于train_utils.py代码的模板，按需修改使用

**可以直接使用有：**

set_seed，checkoutput_and_setcuda，init_logger

**定制的有：**

trainer，evaluation，cal_performance，infer，ModelConfig以及其他应该需要的

### Directly Use

set_seed, checkoutput_and_setcuda, init_logger

```python
import os
import random
import shutil
import logging
import numpy as np
from sklearn.metrics import f1_score

import torch
import torch.nn.functional as F
from tensorboardX import SummaryWriter

from source.utils.engine import Trainer

def set_seed(args):
    random.seed(args.seed)
    np.random.seed(args.seed)
    torch.manual_seed(args.seed)
    if args.n_gpu > 0:
        torch.cuda.manual_seed_all(args.seed)


def checkoutput_and_setcuda(args):
    if not os.path.exists(args.output_dir):
        os.makedirs(args.output_dir)

    if (
        os.path.exists(args.output_dir)
        and os.listdir(args.output_dir)
        and args.do_train
        and not args.overwrite_output_dir
    ):
        raise ValueError(
            f"Output directory ({args.output_dir}) already exists and is not empty. Use --overwrite_output_dir to overcome."
        )

    # Setup CUDA, GPU & distributed training
    if args.local_rank == -1 or args.no_cuda:
        device = torch.device("cuda" if torch.cuda.is_available() and not args.no_cuda else "cpu")
        args.n_gpu = 0 if args.no_cuda else torch.cuda.device_count()
    else:  # Initializes the distributed backend which will take care of sychronizing nodes/GPUs
        torch.cuda.set_device(args.local_rank)
        device = torch.device("cuda", args.local_rank)
        torch.distributed.init_process_group(backend="nccl")
        args.n_gpu = 1
    args.device = device
    return args


def init_logger(args):
    logger = logging.getLogger(__name__)
    logging.basicConfig(
        format="%(asctime)s - %(levelname)s - %(name)s -   %(message)s",
        datefmt="%m/%d/%Y %H:%M:%S",
        level=logging.INFO if args.local_rank in [-1, 0] else logging.WARN,
    )
    logger.warning(
        "Process rank: %s, device: %s, n_gpu: %s, distributed training: %s, 16-bits training: %s",
        args.local_rank,
        args.device,
        args.n_gpu,
        bool(args.local_rank != -1),
        args.fp16,
    )
    return logger

```

### Costomize

#### trainer

定制的部分在于换一下数据读入的方式、loss的计算方式、save strategy以及load strategy。

**TODO**：

- optimizer学习率衰减策略`lr_scheduler`
- Early stop
- `save_summary`可视化
- model`load`函数

```python
class trainer(Trainer):
    def __init__(self, args, model, optimizer, train_iter, valid_iter, logger, valid_metric_name="-loss", save_dir=None,
                 num_epochs=5, log_steps=None, valid_steps=None, grad_clip=None, lr_scheduler=None, save_summary=False):

        super().__init__(args, model, optimizer, train_iter, valid_iter, logger, valid_metric_name, num_epochs,
                         save_dir, log_steps, valid_steps, grad_clip, lr_scheduler, save_summary)

        self.args = args
        self.model = model
        self.optimizer = optimizer
        self.train_iter = train_iter
        self.valid_iter = valid_iter
        self.logger = logger

        self.is_decreased_valid_metric = valid_metric_name[0] == "-"
        self.valid_metric_name = valid_metric_name[1:]
        self.num_epochs = num_epochs
        self.save_dir = save_dir if save_dir else self.args.output_dir
        self.log_steps = log_steps
        self.valid_steps = valid_steps
        self.grad_clip = grad_clip
        self.lr_scheduler = lr_scheduler
        self.save_summary = save_summary

        if self.save_summary:
            self.train_writer = SummaryWriter(
                os.path.join(self.save_dir, "logs", "train"))
            self.valid_writer = SummaryWriter(
                os.path.join(self.save_dir, "logs", "valid"))

        self.best_valid_metric = float("inf") if self.is_decreased_valid_metric else -float("inf")
        self.epoch = 0
        self.global_step = 0
        self.init_message()

    def train_epoch(self):
        self.epoch += 1
        train_start_message = "Training Epoch - {}".format(self.epoch)
        self.logger.info(train_start_message)

        tr_loss, nb_tr_examples, nb_tr_steps = 0, 0, 0
        for batch_id, batch in enumerate(self.train_iter, 1):
            self.model.train()
			
			# 定义自己的输入格式
            inputs_id, inputs_label, _ = tuple(t.to(self.args.device) for t in batch)
            pred = self.model(inputs_id)
            loss = F.cross_entropy(pred, inputs_label)

            if self.args.n_gpu > 1:
                loss = loss.mean()  # mean() to average on multi-gpu.
            if self.args.gradient_accumulation_steps > 1:
                loss = loss / self.args.gradient_accumulation_steps

            if self.args.fp16:
                try:
                    from apex import amp
                except ImportError:
                    raise ImportError(
                        "Please install apex from https://www.github.com/nvidia/apex to use fp16 training.")
                with amp.scale_loss(loss, self.optimizer) as scaled_loss:
                    scaled_loss.backward()
                if self.grad_clip:
                    torch.nn.utils.clip_grad_norm_(amp.master_params(self.optimizer), self.grad_clip)
            else:
                loss.backward()
                if self.grad_clip:
                    torch.nn.utils.clip_grad_norm_(self.model.parameters(), self.grad_clip)

            tr_loss += loss.item()
            nb_tr_examples += inputs_id.size(0)
            nb_tr_steps += 1

            if (batch_id + 1) % self.args.gradient_accumulation_steps == 0:
                self.optimizer.step()
                if self.lr_scheduler:
                    self.lr_scheduler.step()  # Update learning rate schedule
                self.model.zero_grad()
                self.global_step += 1

            if self.global_step % self.log_steps == 0:
                # logging_loss = tr_loss / self.global_step
                self.logger.info("the current train_steps is {}".format(self.global_step))
                self.logger.info("the current logging_loss is {}".format(loss.item()))
			
            ## TODO：提前终止后续安排
            if self.global_step % self.valid_steps == 0:
                self.logger.info(self.valid_start_message)
                self.model.to(self.args.device)
                metrics = evaluate(self.args, self.model, self.valid_iter, self.logger)
                cur_valid_metric = metrics[self.valid_metric_name]
                if self.is_decreased_valid_metric:
                    is_best = cur_valid_metric < self.best_valid_metric
                else:
                    is_best = cur_valid_metric > self.best_valid_metric
                if is_best:
                    self.best_valid_metric = cur_valid_metric
                self.save(is_best)
                self.logger.info("-" * 85 + "\n")


    def train(self):
        if self.args.max_steps > 0:
            t_total = self.args.max_steps
            self.args.num_train_epochs = self.args.max_steps // (len(self.train_iter) // self.args.gradient_accumulation_steps) + 1
        else:
            t_total = len(self.train_iter) // self.args.gradient_accumulation_steps * self.args.num_train_epochs

        self.logger.info(self.train_start_message)
        self.logger.info("Num examples = %d", len(self.train_iter))
        self.logger.info("Num Epochs = %d", self.num_epochs)
        self.logger.info("Instantaneous batch size per GPU = %d", self.args.per_gpu_train_batch_size)
        self.logger.info(
            "Total train batch size (w. parallel, distributed & accumulation) = %d",
            self.args.train_batch_size
            * self.args.gradient_accumulation_steps
            * (torch.distributed.get_world_size() if self.args.local_rank != -1 else 1),
        )
        self.logger.info("Gradient Accumulation steps = %d", self.args.gradient_accumulation_steps)
        self.logger.info("Total optimization steps = %d", t_total)
        self.logger.info("logger steps = %d", self.log_steps)
        self.logger.info("valid steps = %d", self.valid_steps)
        self.logger.info("-" * 85 + "\n")
        if self.args.fp16:
            try:
                from apex import amp
            except ImportError:
                raise ImportError("Please install apex from https://www.github.com/nvidia/apex to use fp16 training.")
            self.model, self.optimizer = amp.initialize(self.model, self.optimizer, opt_level=self.args.fp16_opt_level)

        # multi-gpu training (should be after apex fp16 initialization)
        if self.args.n_gpu > 1:
            self.model = torch.nn.DataParallel(self.model)

        # Distributed training (should be after apex fp16 initialization)
        if self.args.local_rank != -1:
            self.model = torch.nn.parallel.DistributedDataParallel(
                self.model, device_ids=[self.args.local_rank], output_device=self.args.local_rank, find_unused_parameters=True,
            )

        for _ in range(int(self.num_epochs)):
            self.train_epoch()

    def save(self, is_best=False, save_mode="best"):
        ## 可以修改save strategy
        model_file_name = "state_epoch_{}.model".format(self.epoch) if save_mode == "all" else "state.model"
        model_file = os.path.join(
            self.save_dir, model_file_name)
        torch.save(self.model.state_dict(), model_file)
        self.logger.info("Saved model state to '{}'".format(model_file))

        train_file_name = "state_epoch_{}.train".format(self.epoch) if save_mode == "all" else "state.train"
        train_file = os.path.join(
            self.save_dir, train_file_name)
        train_state = {"epoch": self.epoch,
                       "batch_num": self.batch_num,
                       "best_valid_metric": self.best_valid_metric,
                       "settings": self.args}
        torch.save(train_state, train_file)
        self.logger.info("Saved train state to '{}'".format(train_file))

        if is_best:
            best_model_file = os.path.join(self.save_dir, "best.model")
            best_train_file = os.path.join(self.save_dir, "best.train")
            shutil.copy(model_file, best_model_file)
            shutil.copy(train_file, best_train_file)
            self.logger.info(
                "Saved best model state to '{}' with new best valid metric {}-{:.3f}".format(
                    best_model_file, self.valid_metric_name.upper(), self.best_valid_metric))

    def load(self, file_prefix):
        pass
```

#### evaluate

定制的部分在于换一下数据读入的方式、`loss`的计算方式、预测值`pred`的收集方式，`metric`计算方式。

**TODO**：

- 统一的`metric`计算方式，目前比较混乱

```python
def evaluate(args, model, valid_dataset, logger):
    eval_loss, nb_eval_steps = 0.0, 0
    labels, preds = None, None
    model.eval()
    for batch in valid_dataset:
        ## 数据读入方式以及loss的计算方式
        inputs_id, inputs_label, _ = tuple(t.to(args.device) for t in batch)
        with torch.no_grad():
            logits = model(inputs_id)
            tmp_eval_loss = F.cross_entropy(logits, inputs_label)
            if args.n_gpu > 1:
                tmp_eval_loss = tmp_eval_loss.mean()  # mean() to average on multi-gpu parallel evaluating

            eval_loss += tmp_eval_loss.item()
        nb_eval_steps += 1
		
        ## 预测值的收集方式
        if preds is None:
            preds = logits.detach().cpu().numpy()
            labels = inputs_label.detach().cpu().numpy()
        else:
            preds = np.append(preds, logits.detach().cpu().numpy(), axis=0)
            labels = np.append(labels, inputs_label.detach().cpu().numpy(), axis=0)
	
    ## 预测值与真实值计算的方式
    eval_loss = eval_loss / nb_eval_steps
    preds = np.argmax(preds, axis=1)
    metrics = cal_performance(preds, labels)
    metrics.update({"loss": eval_loss})

    for key in sorted(metrics.keys()):
        logger.info("  %s = %s", key.upper(), str(metrics[key]))
    return metrics
```

#### cal_performance

定制在于引入并计算所需要的`metric`，返回的是字典。

**TODO：**

- 考虑是否需要使用MetricManager类

```python
def cal_performance(preds, labels):
    ## 定制所需的metric计算方式以及收集方式
    assert len(preds) == len(labels)
    acc = (preds == labels).mean()
    f1 = f1_score(y_true=labels, y_pred=preds, average="micro")
    mertrics = {"acc": acc, "f1":f1}
    return mertrics
```

#### inference

**ongoing.....**

#### ModelConfig

在加载BasicConfig之后，读入改`model`必要的参数

```python
def ModelConfig(parser):
    # 添加需要的参数
    parser.add_argument("--what_you_want", type=int, default=None)
    args, _ = parser.parse_known_args()
    return args
```

