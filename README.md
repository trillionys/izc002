import streamlit as st
import sqlite3
from datetime import date

st.set_page_config(
    page_title="Automatic Task Manager",
    page_icon="✅",
    layout="centered"
)

conn = sqlite3.connect("workflow.db")
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS rules (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    situation TEXT,
    step_order INTEGER,
    next_task TEXT,
    task_weekday TEXT
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS work_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    work_date TEXT,
    weekday TEXT,
    situation TEXT,
    task TEXT,
    is_done TEXT
)
""")

conn.commit()

st.title("Automatic Task Manager")

weekdays = ["월", "화", "수", "목", "금", "토", "일", "요일 무관"]

menu = st.sidebar.selectbox(
    "메뉴 선택",
    ["업무 상황 등록", "상황 입력 / 업무 추천", "날짜별 업무 기록"] + weekdays
)

if menu == "업무 상황 등록":

    st.header("업무 상황 등록")

    situation = st.text_input("어떤 상황이 발생했을 때?")

    st.subheader("후속 업무 조치")
    st.write("각 후속 업무마다 실행 요일을 여러 개 선택할 수 있습니다.")

    tasks_input = []

    for i in range(1, 6):
        st.markdown(f"### {i}단계")

        task = st.text_input(
            f"{i}단계 후속 업무",
            key=f"task{i}"
        )

        selected_days = st.multiselect(
            f"{i}단계 실행 요일",
            weekdays,
            key=f"days{i}"
        )

        tasks_input.append((i, task, selected_days))

    if st.button("규칙 저장"):

        if situation.strip() == "":
            st.warning("상황을 입력해주세요.")

        else:
            saved_count = 0

            for step_order, task, selected_days in tasks_input:

                if task.strip() != "":

                    if len(selected_days) == 0:
                        st.warning(f"{step_order}단계 업무의 요일을 하나 이상 선택해주세요.")
                        continue

                    for task_weekday in selected_days:
                        cursor.execute(
                            """
                            INSERT INTO rules
                            (situation, step_order, next_task, task_weekday)
                            VALUES (?, ?, ?, ?)
                            """,
                            (situation, step_order, task, task_weekday)
                        )
                        saved_count += 1

            conn.commit()

            if saved_count > 0:
                st.success("업무 상황이 저장되었습니다.")
            else:
                st.warning("저장된 후속 업무가 없습니다.")

    st.subheader("등록된 업무 목록")

    cursor.execute("""
    SELECT id, situation, step_order, next_task, task_weekday
    FROM rules
    ORDER BY situation, step_order, task_weekday
    """)

    rules = cursor.fetchall()

    if rules:
        delete_rule_ids = []

        for r in rules:
            rule_id = r[0]
            situation_name = r[1]
            step_order = r[2]
            next_task = r[3]
            task_weekday = r[4]

            st.markdown("---")

            st.write(
                f"{situation_name} → {step_order}단계. "
                f"{next_task} / 실행 요일: {task_weekday}"
            )

            delete_check = st.checkbox(
                "이 업무 삭제 선택",
                key=f"delete_rule_{rule_id}"
            )

            if delete_check:
                delete_rule_ids.append(rule_id)

        if st.button("선택한 업무 삭제"):
            if len(delete_rule_ids) == 0:
                st.warning("삭제할 업무를 선택해주세요.")
            else:
                for rule_id in delete_rule_ids:
                    cursor.execute(
                        """
                        DELETE FROM rules
                        WHERE id = ?
                        """,
                        (rule_id,)
                    )

                conn.commit()
                st.success("선택한 업무가 삭제되었습니다.")
                st.rerun()
    else:
        st.info("등록된 업무가 없습니다.")

elif menu == "상황 입력 / 업무 추천":

    st.header("상황 입력 / 업무 추천")

    input_situation = st.text_input("현재 발생한 상황을 입력하세요")

    if st.button("후속 업무 추천"):

        cursor.execute(
            """
            SELECT step_order, next_task, task_weekday
            FROM rules
            WHERE situation LIKE ?
            ORDER BY step_order, task_weekday
            """,
            (f"%{input_situation}%",)
        )

        st.session_state.search_results = cursor.fetchall()
        st.session_state.search_situation = input_situation

    if "search_results" in st.session_state:

        results = st.session_state.search_results
        saved_situation = st.session_state.search_situation

        if results:
            st.subheader("추천 후속 업무 체크리스트")

            shown_items = set()

            for task in results:
                step_order = task[0]
                next_task = task[1]
                task_weekday = task[2]

                item_key = (step_order, next_task, task_weekday)

                if item_key in shown_items:
                    continue

                shown_items.add(item_key)

                key = f"search_{saved_situation}_{task_weekday}_{step_order}_{next_task}"

                checked = st.checkbox(
                    f"{step_order}단계. [{task_weekday}] {next_task}",
                    key=key
                )

                if checked:
                    st.markdown(
                        f"<span style='color:gray; text-decoration:line-through;'>"
                        f"{step_order}단계. [{task_weekday}] {next_task}"
                        f"</span>",
                        unsafe_allow_html=True
                    )

        else:
            st.warning("해당 상황에 등록된 후속 업무가 없습니다.")

elif menu == "날짜별 업무 기록":

    st.header("날짜별 업무 기록")

    selected_date = st.date_input("날짜 선택", date.today())

    selected_weekday = st.selectbox(
        "오늘은 무슨 요일인가요?",
        weekdays[:-1]
    )

    st.subheader("오늘 어떤 상황이 있었나요?")

    today_situation = st.text_input(
        "상황 입력",
        placeholder="예: 계약서 수령, 고객 문의, 입금 확인"
    )

    if st.button("오늘 할 일 추천"):

        cursor.execute(
            """
            SELECT situation, step_order, next_task, task_weekday
            FROM rules
            WHERE task_weekday = ? OR task_weekday = '요일 무관'
            ORDER BY situation, step_order
            """,
            (selected_weekday,)
        )

        st.session_state.today_tasks = cursor.fetchall()
        st.session_state.today_date = str(selected_date)
        st.session_state.today_weekday = selected_weekday

    if "today_tasks" in st.session_state:

        recommended_tasks = st.session_state.today_tasks

        if recommended_tasks:
            st.subheader(f"{st.session_state.today_weekday} 추천 업무 리스트")

            shown_today_items = set()

            for task in recommended_tasks:
                situation = task[0]
                step_order = task[1]
                next_task = task[2]

                item_key = (situation, step_order, next_task)

                if item_key in shown_today_items:
                    continue

                shown_today_items.add(item_key)

                key = f"log_{st.session_state.today_date}_{situation}_{step_order}_{next_task}"

                checked = st.checkbox(
                    f"[{situation}] {step_order}단계. {next_task}",
                    key=key
                )

                if checked:
                    st.markdown(
                        f"<span style='color:gray; text-decoration:line-through;'>"
                        f"[{situation}] {step_order}단계. {next_task}"
                        f"</span>",
                        unsafe_allow_html=True
                    )

        else:
            st.info("해당 요일에 등록된 추천 업무가 없습니다.")

    if st.button("오늘 업무 기록 저장"):

        if "today_tasks" not in st.session_state:
            st.warning("먼저 오늘 할 일 추천을 눌러주세요.")

        else:
            saved_items = set()

            for task in st.session_state.today_tasks:
                situation = task[0]
                step_order = task[1]
                next_task = task[2]

                item_key = (situation, step_order, next_task)

                if item_key in saved_items:
                    continue

                saved_items.add(item_key)

                key = f"log_{st.session_state.today_date}_{situation}_{step_order}_{next_task}"

                cursor.execute(
                    """
                    INSERT INTO work_logs
                    (work_date, weekday, situation, task, is_done)
                    VALUES (?, ?, ?, ?, ?)
                    """,
                    (
                        st.session_state.today_date,
                        st.session_state.today_weekday,
                        situation,
                        next_task,
                        "완료" if st.session_state.get(key, False) else "미완료"
                    )
                )

            conn.commit()
            st.success("오늘 업무 기록이 저장되었습니다.")

    if st.button("상황 기반 업무도 추가 기록"):

        if today_situation.strip() == "":
            st.warning("상황을 입력해주세요.")

        else:
            cursor.execute(
                """
                SELECT situation, step_order, next_task, task_weekday
                FROM rules
                WHERE situation LIKE ?
                ORDER BY step_order
                """,
                (f"%{today_situation}%",)
            )

            situation_tasks = cursor.fetchall()

            if situation_tasks:
                saved_situation_items = set()

                for task in situation_tasks:
                    situation = task[0]
                    next_task = task[2]

                    item_key = (situation, next_task)

                    if item_key in saved_situation_items:
                        continue

                    saved_situation_items.add(item_key)

                    cursor.execute(
                        """
                        INSERT INTO work_logs
                        (work_date, weekday, situation, task, is_done)
                        VALUES (?, ?, ?, ?, ?)
                        """,
                        (
                            str(selected_date),
                            selected_weekday,
                            situation,
                            next_task,
                            "미완료"
                        )
                    )

                conn.commit()
                st.success("상황 기반 추천 업무가 기록되었습니다.")
            else:
                st.warning("해당 상황에 등록된 후속 업무가 없습니다.")

    st.subheader("저장된 날짜별 업무 기록")

    cursor.execute(
        """
        SELECT id, work_date, weekday, situation, task, is_done
        FROM work_logs
        ORDER BY work_date DESC, id DESC
        """
    )

    logs = cursor.fetchall()

    delete_ids = []

    for log in logs:
        log_id = log[0]
        work_date = log[1]
        weekday = log[2]
        situation = log[3]
        task = log[4]
        is_done = log[5]

        st.markdown("---")

        checked = st.checkbox(
            f"{work_date} / {weekday} / 상황: {situation} → {task}",
            value=True if is_done == "완료" else False,
            key=f"saved_log_{log_id}"
        )

        new_status = "완료" if checked else "미완료"

        if new_status != is_done:
            cursor.execute(
                """
                UPDATE work_logs
                SET is_done = ?
                WHERE id = ?
                """,
                (new_status, log_id)
            )
            conn.commit()

        if checked:
            st.markdown("✅ 완료")
        else:
            st.markdown("미완료")

        delete_check = st.checkbox(
            "이 기록 삭제 선택",
            key=f"delete_log_{log_id}"
        )

        if delete_check:
            delete_ids.append(log_id)

    if st.button("선택한 기록 삭제"):

        if len(delete_ids) == 0:
            st.warning("삭제할 기록을 선택해주세요.")
        else:
            for log_id in delete_ids:
                cursor.execute(
                    """
                    DELETE FROM work_logs
                    WHERE id = ?
                    """,
                    (log_id,)
                )

            conn.commit()
            st.success("선택한 기록이 삭제되었습니다.")
            st.rerun()

else:

    selected_weekday = menu

    st.header(f"{selected_weekday} 추천 업무")

    cursor.execute(
        """
        SELECT situation, step_order, next_task
        FROM rules
        WHERE task_weekday = ?
        ORDER BY situation, step_order
        """,
        (selected_weekday,)
    )

    tasks = cursor.fetchall()

    if tasks:
        current_situation = ""

        for task in tasks:
            situation = task[0]
            step_order = task[1]
            next_task = task[2]

            if situation != current_situation:
                st.markdown(f"### 상황: {situation}")
                current_situation = situation

            key = f"weekday_{selected_weekday}_{situation}_{step_order}_{next_task}"

            checked = st.checkbox(
                f"{step_order}단계. {next_task}",
                key=key
            )

            if checked:
                st.markdown(
                    f"<span style='color:gray; text-decoration:line-through;'>"
                    f"{step_order}단계. {next_task}"
                    f"</span>",
                    unsafe_allow_html=True
                )

    else:
        st.info(f"{selected_weekday}에 등록된 업무가 없습니다.")

conn.close()
