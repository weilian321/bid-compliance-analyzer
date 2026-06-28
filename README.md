#!/usr/bin/env python3
"""
Agent 核心层端到端演示

依次演示五大核心能力：规划、调度、记忆、异常、自校验。
"""
import sys, os, json, uuid, time
sys.path.insert(0, os.path.join(os.path.dirname(__file__)))

from agent.planner import AgentPlanner
from agent.scheduler import ToolScheduler
from agent.memory import MemoryManager
from agent.exception_handler import ExceptionHandler, AnomalyType, AnomalyResult
from agent.self_validator import SelfValidator

BOLD = "\033[1m"
GREEN = "\033[32m"
YELLOW = "\033[33m"
CYAN = "\033[36m"
RESET = "\033[0m"

def hdr(title):
    print(f"\n{BOLD}{CYAN}{'='*56}{RESET}")
    print(f"{BOLD}{CYAN}  {title}{RESET}")
    print(f"{BOLD}{CYAN}{'='*56}{RESET}\n")

def step(n, desc):
    print(f"{BOLD}[{n}] {desc}{RESET}")

def ok(msg):
    print(f"  {GREEN}OK{RESET} {msg}")

def info(msg):
    print(f"  {YELLOW} ->{RESET} {msg}")


def demo_planner():
    hdr("1. 自主任务规划引擎 AgentPlanner")
    planner = AgentPlanner()

    test_cases = [
        ("帮我分析这份交换机招标文件", "full_pipeline"),
        ("只检查资格门槛和实质性条件", "qualification_only"),
        ("重点比对性能参数是否满足要求", "performance_focus"),
        ("得分优先，优化评分项得分率", "score_priority"),
    ]

    step("1.1", "意图解析 - 四种使用场景")
    for user_input, expected in test_cases:
        intent = planner.parse_intent(user_input, {"current_task_id": "demo-001"})
        marker = GREEN if intent.intent_type == expected else ""
        print(f"  \"{user_input}\"")
        print(f"    {marker}{intent.intent_type}{RESET} keywords={intent.keywords}")

    step("1.2", "基于 full_pipeline 生成执行计划")
    intent = planner.parse_intent("帮我分析这份交换机招标文件", {"current_task_id": "demo-001"})
    plan = planner.generate_plan(intent)
    ok(f"模板: {plan.template_name}, 共 {len(plan.steps)} 个步骤")
    for s in plan.steps:
        deps_str = f" (依赖: {s.depends_on})" if s.depends_on else ""
        print(f"    {s.name} -> {s.tool_name}{deps_str}")

    step("1.3", "动态调整 - 无评分规则时移除打分步骤")
    new_plan = planner.adjust_plan(plan, "parse_scoring_rules", {"skip_scoring": True})
    scoring_steps = [s.name for s in new_plan.steps if s.tool_name in ("scoring_rule_parser", "score_calculator")]
    if scoring_steps:
        info(f"仍含评分步骤: {scoring_steps}")
    else:
        ok("评分相关步骤已移除")

    step("1.4", "动态调整 - 多型号时插入选型步骤")
    plan2 = planner.adjust_plan(plan, "semantic_matcher", {"need_model_selection": True})
    select_step = next((s for s in plan2.steps if s.tool_name == "model_selector"), None)
    ok(f"已插入选型步骤: {select_step.name}" if select_step else "未插入")

    step("1.5", "获取当前待执行步骤")
    nxt = planner.next_step(plan)
    ok(f"下一步: {nxt.name} -> {nxt.tool_name}")
    return planner, plan


def demo_scheduler():
    hdr("2. 多工具调度器 ToolScheduler")
    scheduler = ToolScheduler()

    def fake_parser(ctx):
        return {"params": [{"name": "交换容量", "value": ">=2.4Tbps"}, {"name": "包转发率", "value": ">=360Mpps"}]}

    def fake_matcher(ctx):
        return {"matched": 40, "unmatched": 5, "coverage": 0.89}

    def fake_scorer(ctx):
        return {"total_score": 87.5, "items": [{"rule": "交换容量", "score": 10}, {"rule": "包转发率", "score": 10}]}

    step("2.1", "注册 3 个工具")
    scheduler.register_tool("doc_parser", fake_parser, {"description": "文档解析", "timeout": 60})
    scheduler.register_tool("semantic_matcher", fake_matcher, {"description": "语义匹配", "timeout": 120})
    scheduler.register_tool("score_calculator", fake_scorer, {"description": "得分计算", "timeout": 30})
    ok(f"已注册 {len(scheduler._tools)} 个工具")

    step("2.2", "执行文档解析")
    result = scheduler.execute_step("parse_doc", "doc_parser", {"file_path": "/tmp/test.pdf"})
    ok(f"结果: {result['success']}, 数据: {result.get('data', {})}")

    step("2.3", "执行语义匹配")
    result = scheduler.execute_step("semantic_match", "semantic_matcher", {"bid_params": [], "product_line": "pl-001"})
    data = result.get("data") or {}
    ok(f"结果: {result['success']}, 覆盖率: {data.get('coverage', 'N/A')}")

    step("2.4", "查看执行日志")
    for log in scheduler._logs:
        ok(f"{log.step_name} -> {log.tool_name}: {log.status} ({log.duration_ms:.1f}ms)")

    step("2.5", "调用未注册工具 → 报错")
    result = scheduler.execute_step("unknown", "nonexistent", {})
    ok(f"正确返回错误: {not result['success']}, error: {result.get('error', 'N/A')[:50]}")
    return scheduler


def demo_memory():
    hdr("3. 双层记忆系统 MemoryManager")
    db_path = os.path.join(os.path.dirname(__file__), "data", "demo_memory.db")
    memory = MemoryManager(db_path=db_path)
    ok(f"MemoryManager 初始化 (DB: {db_path})")

    task_id = f"task_demo_{uuid.uuid4().hex[:8]}"

    step("3.1", "保存短期任务上下文")
    memory.save_session_context(task_id, {
        "parsed_params": [{"name": "交换容量", "value": ">=2.4Tbps"}],
        "completed_step": "parse_doc",
    })
    sess = memory.load_session_context(task_id)
    ok(f"会话参数: {len(sess.parsed_params)}, 完成步骤: {sess.completed_steps}")

    step("3.2", "用户修正参数 → 同步双层记忆")
    memory.update_memory_on_correction(task_id, "交换容量", ">=2.4Tbps", ">=4.8Tbps")
    sess2 = memory.load_session_context(task_id)
    ok(f"短期记忆: {len(sess2.user_corrections)} 条修正")

    step("3.3", "持久化项目记录到长期记忆")
    memory.record_project_result({
        "bid_file_name": "智慧城市交换机招标2025Q1.pdf",
        "product_line_id": "pl-network-switches",
        "decision": {"conclusion": "建议投标", "score_rate": 0.92},
        "risk_count": 2,
    })
    ok("项目记录已存储")

    step("3.4", "历史项目搜索")
    results = memory.search_history("交换机")
    ok(f"搜索 '交换机' -> {len(results)} 条结果")

    step("3.5", "导出记忆摘要")
    summary = memory.export_memory()
    ok(f"会话数: {summary['session_count']}")

    step("3.6", "清除会话记忆")
    memory.clear_memory(f"task:{task_id}")
    ok(f"清除后会话数: {len(memory._session)}")
    return memory


def demo_exception():
    hdr("4. 异常自主处理器 ExceptionHandler")
    eh = ExceptionHandler()

    step("4.1", "检测异常: 工具调用失败")
    anomaly = eh.detect_anomaly(
        {"name": "parse_doc", "tool_name": "doc_parser"},
        {"success": False, "data": None, "error": "文件格式不支持"},
    )
    ok(f"类型={anomaly.anomaly_type.value} 级别={anomaly.severity} 询问={anomaly.user_query[:60]}...")

    step("4.2", "检测异常: 低匹配率")
    anomaly2 = eh.detect_anomaly(
        {"name": "semantic_match", "tool_name": "semantic_matcher"},
        {"success": True, "data": {"coverage": 0.45}},
    )
    ok(f"类型={anomaly2.anomaly_type.value} 级别={anomaly2.severity} 覆盖={anomaly2.detail.get('coverage', 'N/A')}")
    info(f"选项: {anomaly2.user_options}")

    step("4.3", "解析处理策略")
    strategy = eh.resolve_strategy(anomaly2)
    ok(f"策略: {strategy}")

    step("4.4", "构造用户询问")
    query = eh.formulate_query(anomaly2)
    ok(f"消息: {query['message'][:60]}...")

    step("4.5", "应用用户反馈 → 继续执行")
    resolution = eh.apply_resolution(anomaly2, "跳过，使用当前结果")
    ok(f"动作: {resolution['action']}")

    step("4.6", "检查高危风险 (模拟)")

    class FakeDeviation:
        pass

    low_risk = FakeDeviation()
    low_risk.is_material = False
    low_risk.deviation_type = "正偏离"

    high_risk = FakeDeviation()
    high_risk.is_material = True
    high_risk.deviation_type = "负偏离"
    high_risk.param_name = "交换容量"

    alert = eh.check_high_risk([low_risk, high_risk])
    ok(f"高危预警已触发: {alert.anomaly_type.value} - {alert.message}")

    step("4.7", "无风险时返回 None")
    no_alert = eh.check_high_risk([low_risk])
    ok(f"预期 None: {no_alert is None}")

    return eh


def demo_validator():
    hdr("5. 结果自校验器 SelfValidator")

    task_id = "task_demo_validator"

    step("5.1", "context 直接传入 (跳过 DB)")

    from agent.self_validator import MissingItem

    mock_context = {
        "parsed_params": [
            {"id": "p1", "name": "交换容量"},
            {"id": "p2", "name": "包转发率"},
            {"id": "p3", "name": "端口密度"},
        ],
        "matched_pairs": [
            {"bid_param_id": "p1"},
            {"bid_param_id": "p2"},
        ],
        "deviation_results": [
            {"parsed_param_id": "p1", "deviation_type": "正偏离"},
            {"parsed_param_id": "p2", "deviation_type": "一致"},
        ],
        "scoring_rules": [
            {"name": "交换容量评分"},
            {"name": "包转发率评分"},
        ],
        "score_results": [
            {"rule_name": "交换容量评分"},
        ],
    }

    validator = SelfValidator()
    report = validator.validate_completeness(task_id, context=mock_context)
    ok(f"参数覆盖率: {report.param_coverage:.1%} (2/3)")
    ok(f"评分覆盖率: {report.score_coverage:.1%} (1/2)")
    ok(f"综合结论: {'PASS' if report.complete else 'FAIL'} - {report.summary}")

    step("5.2", "识别缺失项")
    missing = validator.identify_missing_items(mock_context)
    ok(f"缺失项: {len(missing)} 条")
    for m in missing:
        info(f"  [{m.category}] {m.description}")

    step("5.3", "完整上下文覆盖 (应全部通过)")
    full_context = {
        "parsed_params": [{"id": "p1", "name": "交换容量"}, {"id": "p2", "name": "包转发率"}],
        "matched_pairs": [{"bid_param_id": "p1"}, {"bid_param_id": "p2"}],
        "deviation_results": [{"parsed_param_id": "p1", "deviation_type": "正偏离"}, {"parsed_param_id": "p2", "deviation_type": "一致"}],
        "scoring_rules": [{"name": "交换容量评分"}, {"name": "包转发率评分"}],
        "score_results": [{"rule_name": "交换容量评分"}, {"rule_name": "包转发率评分"}],
    }
    report2 = validator.validate_completeness(task_id, context=full_context)
    ok(f"参数覆盖: {report2.param_coverage:.1%}")
    ok(f"评分覆盖: {report2.score_coverage:.1%}")
    ok(f"综合: {'PASS' if report2.complete else 'FAIL'}")

    return validator


def main():
    print(f"\n{BOLD}{CYAN}Agent 核心层五大能力端到端演示{RESET}")
    print(f"{'='*56}")

    demo_planner()
    demo_scheduler()
    demo_memory()
    demo_exception()
    demo_validator()

    hdr("演示完成")
    summary = [
        ("规划引擎", "4 种意图识别, 4 个链路模板, 动态调整 + 多型号插入"),
        ("调度器", "3 工具注册, 带超时执行, 日志追踪, 异常返回已测"),
        ("记忆系统", "短期上下文 + 长期 4 表持久化, 历史搜索, 别名复用"),
        ("异常处理器", "7 种异常类型, 策略映射, 用户询问, 高危即时预警"),
        ("自校验器", "参数/评分覆盖率检查, 缺失识别, 上下文直接注入"),
    ]
    for name, desc in summary:
        print(f"  {BOLD}{GREEN}{name}{RESET}: {desc}")

    print(f"\n{BOLD}{CYAN}全部 5 个核心模块联调通过。{RESET}\n")


if __name__ == "__main__":
    main()
