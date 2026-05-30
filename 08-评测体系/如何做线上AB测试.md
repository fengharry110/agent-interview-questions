# 如何做线上AB测试？

## 面试问题

如何做线上AB测试？

## 考察重点

- 是否理解智能体系统的 A/B 测试和传统 Web A/B 测试的差异
- 是否能说清楚 A/B 测试的流量分割和评估方法
- 是否了解智能体 A/B 测试的挑战（非确定性输出、长尾延迟）
- 是否有过智能体 A/B 测试的实际经验

## 核心结论

智能体系统的 A/B 测试比传统 Web A/B 测试更复杂，因为输出不是固定的 HTML 页面而是动态的决策链。核心方法是：**按用户/会话粒度分流**、**用统一指标体系评估**、**跑足够长的时间消除随机性**。关键原则是：不要同时跑太多实验（互斥 + 样本量不足），不要提前下结论（智能体的表现波动大）。

## 参考回答

### A/B 测试设计

```python
class AgentABTest:
    """
    智能体 A/B 测试
    """
    
    def __init__(self):
        self.traffic_split = TrafficSplitter()
        self.metrics_collector = MetricsCollector()
    
    def setup_experiment(self, config: ABTestConfig) -> Experiment:
        """
        设置 A/B 测试实验
        """
        experiment = Experiment(
            name=config.name,
            control=config.control_version,
            treatment=config.treatment_version,
            traffic_split=config.traffic_split,
            duration_days=config.duration_days,
            min_sample_size=config.min_sample_size
        )
        
        # 注册分流规则
        self.traffic_split.register_experiment(
            experiment.name,
            {
                "control": config.traffic_split['control'],
                "treatment": config.traffic_split['treatment']
            }
        )
        
        return experiment
    
    def analyze_results(
        self,
        experiment: Experiment,
        days: int
    ) -> ABTestResult:
        """
        分析 A/B 测试结果
        """
        control_data = self.metrics_collector.collect(
            experiment='control',
            days=days
        )
        treatment_data = self.metrics_collector.collect(
            experiment='treatment',
            days=days
        )
        
        return ABTestResult(
            experiment_name=experiment.name,
            duration_days=days,
            sample_size={
                'control': len(control_data),
                'treatment': len(treatment_data)
            },
            metrics=self._compare_metrics(
                control_data, treatment_data
            ),
            significance=self._calculate_significance(
                control_data, treatment_data
            ),
            recommendation=self._generate_recommendation(
                control_data, treatment_data
            )
        )
```

### 流量分割策略

```python
class TrafficSplitter:
    """
    流量分割器
    """
    
    def __init__(self):
        self.experiments: Dict[str, ExperimentConfig] = {}
    
    def assign_group(self, user_id: str) -> str:
        """
        根据用户 ID 分配实验组
        使用一致性哈希保证同一用户始终在同一组
        """
        hash_val = hash(user_id) % 100
        for exp_name, config in self.experiments.items():
            if hash_val < config['control']:
                return f"{exp_name}:control"
            elif hash_val < config['control'] + config['treatment']:
                return f"{exp_name}:treatment"
        return "default"
    
    def register_experiment(
        self,
        name: str,
        split: Dict[str, int]
    ):
        """
        注册实验
        split 示例: {"control": 50, "treatment": 50}
        """
        assert split['control'] + split['treatment'] == 100
        self.experiments[name] = split
```

## 深度展开

### 智能体 A/B 测试的挑战

```python
class ABTestChallenges:
    """
    智能体 A/B 测试的特殊挑战
    """
    
    CHALLENGES = [
        {
            "title": "非确定性输出",
            "description": "同样的输入，智能体的输出可能不同",
            "impact": "需要更大的样本量才能检测出显著差异",
            "mitigation": "温度设为 0，增加样本量"
        },
        {
            "title": "长尾延迟",
            "description": "少数任务可能耗时极长（几十步）",
            "impact": "平均延迟被长尾拉高，但中位数正常",
            "mitigation": "同时看 P50、P95、P99"
        },
        {
            "title": "指标关联",
            "description": "完成率和成本负相关",
            "impact": "单一指标提升可能掩盖其他指标恶化",
            "mitigation": "设置复合指标作为主要决策依据"
        },
        {
            "title": "Carryover Effect",
            "description": "前一次执行的结果影响下一次",
            "impact": "同一个用户连续使用可能产生偏差",
            "mitigation": "按用户分组而不是按请求分组"
        }
    ]
```

### 评估指标

```python
class ABTestMetrics:
    """
    A/B 测试评估指标
    """
    
    PRIMARY_METRICS = {
        "task_completion_rate": {
            "type": "primary",
            "description": "任务完成率"
        },
        "user_satisfaction": {
            "type": "primary",
            "description": "用户满意度（点赞率）"
        },
        "cost_per_task": {
            "type": "primary",
            "description": "单任务成本"
        }
    }
    
    SECONDARY_METRICS = {
        "avg_latency": "平均响应时间",
        "tool_accuracy": "工具调用准确率",
        "intervention_rate": "人工介入率",
        "token_efficiency": "Token 效率"
    }
    
    @staticmethod
    def significance_test(
        control: List[float],
        treatment: List[float],
        alpha: float = 0.05
    ) -> TestResult:
        """
        显著性检验
        """
        from scipy import stats
        
        t_stat, p_value = stats.ttest_ind(control, treatment)
        
        return TestResult(
            significant=p_value < alpha,
            p_value=p_value,
            t_statistic=t_stat,
            control_mean=mean(control),
            treatment_mean=mean(treatment),
            lift=(mean(treatment) - mean(control)) / mean(control)
        )
```

### 实验周期

```python
class ExperimentDuration:
    """
    实验持续时间计算
    """
    
    @staticmethod
    def calculate_required_duration(
        baseline_rate: float,
        minimum_detectable_effect: float,
        daily_traffic: int
    ) -> int:
        """
        计算需要的实验天数
        """
        # 简化的样本量计算
        z_alpha = 1.96  # 95% 置信度
        z_beta = 0.84   # 80% 统计功效
        
        p_pooled = baseline_rate * (1 - baseline_rate)
        effect_size = minimum_detectable_effect ** 2
        
        sample_size_per_group = int(
            2 * p_pooled * (z_alpha + z_beta) ** 2 / effect_size
        )
        
        days_needed = math.ceil(
            sample_size_per_group / (daily_traffic / 2)
        )
        
        return max(days_needed, 7)  # 至少跑 7 天
    
    DURATION_RECOMMENDATIONS = {
        "high_traffic": {
            "daily_sessions": "1000+",
            "recommended_days": "7-14"
        },
        "medium_traffic": {
            "daily_sessions": "100-1000",
            "recommended_days": "14-28"
        },
        "low_traffic": {
            "daily_sessions": "< 100",
            "recommended_days": "28+ 或不适合 A/B 测试"
        }
    }
```

## 工程实践

### A/B 测试平台集成

```python
class ABTestPlatform:
    """
    A/B 测试平台
    """
    
    def __init__(self):
        self.experiments: Dict[str, Experiment] = {}
    
    def start_experiment(self, config: ABTestConfig):
        """启动实验"""
        exp = Experiment(config)
        self.experiments[config.name] = exp
        
        # 注册分流
        self.traffic_splitter.register(
            config.name,
            config.traffic_split
        )
        
        # 记录实验元信息
        self.metadata[config.name] = {
            "started_at": datetime.now(),
            "config": config,
            "status": "running"
        }
    
    def stop_experiment(self, name: str):
        """停止实验并分析结果"""
        exp = self.experiments[name]
        
        # 收集数据
        result = self.analyzer.analyze(exp, exp.duration_days)
        
        # 记录
        self.metadata[name]["status"] = "completed"
        self.metadata[name]["result"] = result
        
        # 清理分流规则
        self.traffic_splitter.remove(name)
        
        return result
    
    def get_running_experiments(self) -> List[str]:
        """获取正在运行的实验"""
        return [
            name for name, meta in self.metadata.items()
            if meta['status'] == 'running'
        ]
```

## 常见误区

- **样本量不够就下结论**：跑了两小时发现 treatment 好了 5% 就宣布胜利。智能体的随机波动很大，需要足够的样本量才能排除随机性
- **同时跑太多实验**：实验之间有交互影响，且分流导致每组样本太少。建议最多同时跑 2-3 个实验
- **只看平均值不看分布**：treatment 可能对 90% 用户更好但对 10% 用户极差。需要看分布而不是只看平均值
- **提前停止实验**：每周检查一次结果，看到显著了就停。多次检查需要做多重假设校正
- **忽略 Novely Effect**：用户对新版本的好奇可能短期拉高满意度。跑满至少一个完整的业务周期

## 追问方向

- 你的 A/B 测试最小 detectable effect 是多少？需要多大的样本量？
- 如何处理"treatment 在 A 指标提升了 3%，但在 B 指标下降了 5%"的 trade-off？
- 如果你只有 100 个用户/天，你还做 A/B 测试吗？替代方案是什么？
- 线上线下结果不一致（离线评测好但线上 A/B 没提升）怎么办？

## 学习建议

从一个简单的实验开始：改一行 prompt（比如改一个工具的描述），然后跑一周 A/B 测试。重点关注"怎么定义评估指标"和"怎么判断显著性"这两个环节。不要一上来就做多变量实验。能从 A/A 测试（对照组 vs 自己）中得到"没有显著差异"的结论，说明你的实验平台搭对了。
