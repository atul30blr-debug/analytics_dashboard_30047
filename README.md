# analytics_dashboard_30047
Code for the frontend, backend, and database

# CODE arrangement 
# 1.Backend code line 8-221
# 2.frontend code line 224-384
# 3.SQL query 389-432

# Backend code

import psycopg2
import pandas as pd

# Database connection details
DB_NAME = "your_db_name"
DB_USER = "your_user"
DB_PASS = "your_password"
DB_HOST = "localhost"
DB_PORT = "5432"

def get_connection():
    """Establishes and returns a new database connection."""
    conn = psycopg2.connect(
        dbname=DB_NAME,
        user=DB_USER,
        password=DB_PASS,
        host=DB_HOST,
        port=DB_PORT
    )
    return conn



# --- User Profile CRUD ---
def create_user(name, email, weight):
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute("INSERT INTO users (name, email, weight_kg) VALUES (%s, %s, %s);", (name, email, weight))
    conn.commit()
    cursor.close()
    conn.close()

def get_user_by_email(email):
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT user_id, name, email, weight_kg FROM users WHERE email = %s;", (email,))
    user = cursor.fetchone()
    cursor.close()
    conn.close()
    return user

def update_user_weight(user_id, weight):
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute("UPDATE users SET weight_kg = %s WHERE user_id = %s;", (weight, user_id))
    conn.commit()
    cursor.close()
    conn.close()

# --- Friends CRUD ---
def add_friend(user_id, friend_email):
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT user_id FROM users WHERE email = %s;", (friend_email,))
    friend_id = cursor.fetchone()
    if friend_id:
        friend_id = friend_id[0]
        cursor.execute("INSERT INTO friends (user_id, friend_id) VALUES (%s, %s);", (user_id, friend_id))
        conn.commit()
        cursor.close()
        conn.close()
        return True
    return False

def get_friends(user_id):
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute("""
        SELECT u.name, u.email 
        FROM friends f
        JOIN users u ON f.friend_id = u.user_id
        WHERE f.user_id = %s;
    """, (user_id,))
    friends = cursor.fetchall()
    cursor.close()
    conn.close()
    return friends

def remove_friend(user_id, friend_email):
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT user_id FROM users WHERE email = %s;", (friend_email,))
    friend_id = cursor.fetchone()
    if friend_id:
        friend_id = friend_id[0]
        cursor.execute("DELETE FROM friends WHERE user_id = %s AND friend_id = %s;", (user_id, friend_id))
        conn.commit()
        cursor.close()
        conn.close()
        return True
    return False

# --- Workout and Exercises CRUD ---
def log_workout(user_id, workout_date, duration):
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO workouts (user_id, workout_date, duration_min) VALUES (%s, %s, %s) RETURNING workout_id;",
        (user_id, workout_date, duration)
    )
    workout_id = cursor.fetchone()[0]
    conn.commit()
    cursor.close()
    conn.close()
    return workout_id

def add_exercise(workout_id, exercise_name, sets, reps, weight):
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO exercises (workout_id, exercise_name, sets, reps, weight) VALUES (%s, %s, %s, %s, %s);",
        (workout_id, exercise_name, sets, reps, weight)
    )
    conn.commit()
    cursor.close()
    conn.close()

def get_workout_history(user_id):
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute("""
        SELECT w.workout_date, w.duration_min, e.exercise_name, e.sets, e.reps, e.weight
        FROM workouts w
        LEFT JOIN exercises e ON w.workout_id = e.workout_id
        WHERE w.user_id = %s
        ORDER BY w.workout_date DESC;
    """, (user_id,))
    history = cursor.fetchall()
    cursor.close()
    conn.close()
    return history

# --- Goals CRUD ---
def create_goal(user_id, description, start_date, target_date):
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO goals (user_id, goal_description, start_date, target_date) VALUES (%s, %s, %s, %s);",
        (user_id, description, start_date, target_date)
    )
    conn.commit()
    cursor.close()
    conn.close()

def get_goals(user_id):
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT goal_id, goal_description, start_date, target_date, is_completed FROM goals WHERE user_id = %s;", (user_id,))
    goals = cursor.fetchall()
    cursor.close()
    conn.close()
    return goals

def update_goal_status(goal_id, is_completed):
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute("UPDATE goals SET is_completed = %s WHERE goal_id = %s;", (is_completed, goal_id))
    conn.commit()
    cursor.close()
    conn.close()

def delete_goal(goal_id):
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute("DELETE FROM goals WHERE goal_id = %s;", (goal_id,))
    conn.commit()
    cursor.close()
    conn.close()

# --- Business Insights ---
def get_insights(user_id):
    conn = get_connection()
    cursor = conn.cursor()

    # Total Workouts
    cursor.execute("SELECT COUNT(*) FROM workouts WHERE user_id = %s;", (user_id,))
    total_workouts = cursor.fetchone()[0]

    # Average Duration
    cursor.execute("SELECT AVG(duration_min) FROM workouts WHERE user_id = %s;", (user_id,))
    avg_duration = cursor.fetchone()[0]

    # Max/Min Duration
    cursor.execute("SELECT MAX(duration_min), MIN(duration_min) FROM workouts WHERE user_id = %s;", (user_id,))
    max_duration, min_duration = cursor.fetchone()

    # Total reps and weight lifted
    cursor.execute("SELECT SUM(reps), SUM(weight * reps) FROM exercises e JOIN workouts w ON e.workout_id = w.workout_id WHERE w.user_id = %s;", (user_id,))
    total_reps, total_weight_lifted = cursor.fetchone()

    # Leaderboard (simplified, based on total workout minutes in the last 7 days)
    cursor.execute("""
        SELECT u.name, SUM(w.duration_min) AS total_duration
        FROM users u
        JOIN workouts w ON u.user_id = w.user_id
        WHERE w.workout_date >= CURRENT_DATE - INTERVAL '7 days'
        AND u.user_id IN (SELECT friend_id FROM friends WHERE user_id = %s) OR u.user_id = %s
        GROUP BY u.name
        ORDER BY total_duration DESC;
    """, (user_id, user_id))
    leaderboard = cursor.fetchall()
    
    cursor.close()
    conn.close()
    
    return {
        "total_workouts": total_workouts,
        "avg_duration": avg_duration,
        "max_duration": max_duration,
        "min_duration": min_duration,
        "total_reps": total_reps,
        "total_weight_lifted": total_weight_lifted,
        "leaderboard": leaderboard
    }

# Frontend code
import streamlit as st
import pandas as pd
from datetime import date
from backend_fta import * # Import all functions from the backend

st.title("💪 Personal Fitness Tracker")

# Session state to manage user login
if 'logged_in_user' not in st.session_state:
    st.session_state.logged_in_user = None

def show_login_page():
    st.header("Login or Register")
    login_email = st.text_input("Enter your email to login")
    if st.button("Login"):
        user = get_user_by_email(login_email)
        if user:
            st.session_state.logged_in_user = {"id": user[0], "name": user[1], "email": user[2], "weight": user[3]}
            st.experimental_rerun()
        else:
            st.error("User not found. Please register.")
            
    st.subheader("New User Registration")
    new_name = st.text_input("Name")
    new_email = st.text_input("Email")
    new_weight = st.number_input("Weight (kg)")
    if st.button("Register"):
        create_user(new_name, new_email, new_weight)
        st.success("Registration successful! You can now log in.")

def show_main_app():
    user = st.session_state.logged_in_user
    st.sidebar.title(f"Welcome, {user['name']}!")
    
    selected_section = st.sidebar.radio(
        "Navigation",
        ["Dashboard", "Log Workout", "Manage Friends", "Set Goals", "Business Insights"]
    )

    if selected_section == "Dashboard":
        st.header("Dashboard")
        st.write(f"**Current Weight:** {user['weight']} kg")
        st.subheader("Workout History")
        history = get_workout_history(user['id'])
        if history:
            df = pd.DataFrame(history, columns=["Date", "Duration (min)", "Exercise", "Sets", "Reps", "Weight"])
            st.dataframe(df)
        else:
            st.info("No workout history yet.")

    elif selected_section == "Log Workout":
        st.header("Log a New Workout")
        with st.form("new_workout_form"):
            workout_date = st.date_input("Date", date.today())
            duration = st.number_input("Duration (minutes)", min_value=1)
            
            st.subheader("Exercises")
            num_exercises = st.number_input("Number of exercises", min_value=1, value=1)
            exercises_data = []
            for i in range(num_exercises):
                st.write(f"**Exercise {i+1}**")
                name = st.text_input(f"Exercise Name {i+1}")
                sets = st.number_input(f"Sets {i+1}", min_value=1)
                reps = st.number_input(f"Reps {i+1}", min_value=1)
                weight = st.number_input(f"Weight (kg) {i+1}", min_value=0.0)
                exercises_data.append((name, sets, reps, weight))

            submitted = st.form_submit_button("Log Workout")
            if submitted:
                workout_id = log_workout(user['id'], workout_date, duration)
                for exercise in exercises_data:
                    add_exercise(workout_id, *exercise)
                st.success("Workout logged successfully!")
                st.experimental_rerun()

    elif selected_section == "Manage Friends":
        st.header("Manage Friends")
        st.subheader("Add a Friend")
        friend_email = st.text_input("Friend's Email")
        if st.button("Add Friend"):
            if add_friend(user['id'], friend_email):
                st.success("Friend added successfully!")
            else:
                st.error("Friend not found or already added.")
        
        st.subheader("Your Friends")
        friends_list = get_friends(user['id'])
        if friends_list:
            df = pd.DataFrame(friends_list, columns=["Name", "Email"])
            st.dataframe(df)
            
            st.subheader("Remove a Friend")
            email_to_remove = st.selectbox("Select friend to remove", [f[1] for f in friends_list])
            if st.button("Remove Friend"):
                if remove_friend(user['id'], email_to_remove):
                    st.success("Friend removed.")
                    st.experimental_rerun()
        else:
            st.info("You have no friends yet.")

    elif selected_section == "Set Goals":
        st.header("Set and Track Goals")
        st.subheader("Create New Goal")
        with st.form("new_goal_form"):
            goal_desc = st.text_area("Goal Description")
            start_date = st.date_input("Start Date", date.today())
            target_date = st.date_input("Target Date")
            submitted = st.form_submit_button("Set Goal")
            if submitted:
                create_goal(user['id'], goal_desc, start_date, target_date)
                st.success("Goal set successfully!")
                st.experimental_rerun()

        st.subheader("Your Goals")
        goals_list = get_goals(user['id'])
        if goals_list:
            for goal in goals_list:
                col1, col2, col3 = st.columns([0.6, 0.2, 0.2])
                with col1:
                    st.markdown(f"**Goal:** {goal[1]}")
                    st.markdown(f"**Status:** {'✅ Completed' if goal[4] else '⏳ In Progress'}")
                with col2:
                    if not goal[4] and st.button("Mark Complete", key=f"complete_{goal[0]}"):
                        update_goal_status(goal[0], True)
                        st.experimental_rerun()
                with col3:
                    if st.button("Delete", key=f"delete_{goal[0]}"):
                        delete_goal(goal[0])
                        st.experimental_rerun()
        else:
            st.info("You have no goals yet.")

    elif selected_section == "Business Insights":
        st.header("Your Fitness Insights")
        insights = get_insights(user['id'])
        
        st.subheader("Workout Summary")
        col1, col2, col3 = st.columns(3)
        col1.metric("Total Workouts", insights["total_workouts"])
        col2.metric("Average Duration", f"{insights['avg_duration']:.2f} min" if insights['avg_duration'] else "N/A")
        col3.metric("Max/Min Duration", f"{insights['max_duration'] or 'N/A'}/{insights['min_duration'] or 'N/A'} min")

        st.subheader("Lifting Progress")
        col1, col2 = st.columns(2)
        col1.metric("Total Reps", insights['total_reps'] or 0)
        col2.metric("Total Weight Lifted", f"{insights['total_weight_lifted'] or 0:.2f} kg")

        st.subheader("Friends' Leaderboard (Last 7 Days)")
        if insights['leaderboard']:
            df = pd.DataFrame(insights['leaderboard'], columns=["Name", "Total Duration (min)"])
            st.dataframe(df)
        else:
            st.info("No leaderboard data available for the last 7 days.")


# Main application entry point
if st.session_state.logged_in_user:
    show_main_app()
else:
    show_login_page()

# SQL QUERY FOR THE SAME

-- DDL for the fitness tracker database

-- Table for user profiles
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    weight_kg DECIMAL(5, 2)
);

-- Table for managing friend connections
CREATE TABLE friends (
    user_id INT REFERENCES users(user_id) ON DELETE CASCADE,
    friend_id INT REFERENCES users(user_id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, friend_id)
);

-- Table for workout sessions
CREATE TABLE workouts (
    workout_id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(user_id) ON DELETE CASCADE,
    workout_date DATE NOT NULL,
    duration_min INT NOT NULL
);

-- Table for individual exercises within a workout
CREATE TABLE exercises (
    exercise_id SERIAL PRIMARY KEY,
    workout_id INT REFERENCES workouts(workout_id) ON DELETE CASCADE,
    exercise_name VARCHAR(255) NOT NULL,
    sets INT,
    reps INT,
    weight DECIMAL(6, 2)
);

-- Table for personal fitness goals
CREATE TABLE goals (
    goal_id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(user_id) ON DELETE CASCADE,
    goal_description TEXT NOT NULL,
    start_date DATE NOT NULL,
    target_date DATE,
    is_completed BOOLEAN DEFAULT FALSE
);
