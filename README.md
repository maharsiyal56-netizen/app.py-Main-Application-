import os
from flask import Flask, render_template, request, redirect, url_for, flash, jsonify
from flask_login import LoginManager, current_user, login_user, logout_user, login_required
from models import db, User, Teacher, Student, Parent, Class, Assignment, Attendance, Grade, Announcement, Event
from config import Config
from datetime import datetime, timedelta
import json
from functools import wraps
import schedule
import threading
import time
from sqlalchemy.exc import IntegrityError

# Create Flask app
app = Flask(__name__)
app.config.from_object(Config)

# Initialize extensions
db.init_app(app)
login_manager = LoginManager(app)
login_manager.login_view = 'auth.login'
login_manager.login_message = 'Please log in to access this page.'

# Role-based access control decorator
def role_required(roles):
    def decorator(f):
        @wraps(f)
        @login_required
        def decorated_function(*args, **kwargs):
            if current_user.role not in roles:
                flash('You do not have permission to access this page.', 'danger')
                return redirect(url_for('dashboard'))
            return f(*args, **kwargs)
        return decorated_function
    return decorator

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

# Automation functions
def check_due_assignments():
    with app.app_context():
        due_soon = Assignment.query.filter(
            Assignment.due_date > datetime.now(),
            Assignment.due_date <= datetime.now() + timedelta(hours=24)
        ).all()
        
        for assignment in due_soon:
            # Get students in the class
            class_students = ClassStudent.query.filter_by(class_id=assignment.class_id).all()
            
            for cs in class_students:
                # Check if student has already submitted
                submission = AssignmentSubmission.query.filter_by(
                    assignment_id=assignment.id, 
                    student_id=cs.student_id
                ).first()
                
                if not submission:
                    # Send reminder (in a real app, this would be an email or notification)
                    print(f"Reminder: Assignment '{assignment.title}' is due soon for student ID {cs.student_id}")

def check_attendance():
    with app.app_context():
        # Mark students absent if they haven't been marked present for today's classes
        today = datetime.now().date()
        classes_today = Class.query.all()  # In a real app, filter by schedule
        
        for class_ in classes_today:
            # Get all students in the class
            class_students = ClassStudent.query.filter_by(class_id=class_.id).all()
            
            for cs in class_students:
                # Check if attendance already recorded for today
                attendance = Attendance.query.filter_by(
                    student_id=cs.student_id,
                    class_id=class_.id,
                    date=today
                ).first()
                
                if not attendance:
                    # Mark as absent
                    new_attendance = Attendance(
                        student_id=cs.student_id,
                        class_id=class_.id,
                        date=today,
                        status='absent'
                    )
                    db.session.add(new_attendance)
        
        db.session.commit()

def run_scheduler():
    while True:
        schedule.run_pending()
        time.sleep(60)

# Schedule tasks
schedule.every().day.at("08:00").do(check_attendance)
schedule.every().day.at("17:00").do(check_due_assignments)

# Start the scheduler in a separate thread
scheduler_thread = threading.Thread(target=run_scheduler)
scheduler_thread.daemon = True
scheduler_thread.start()

# Routes
@app.route('/')
def index():
    if current_user.is_authenticated:
        return redirect(url_for('dashboard'))
    return render_template('index.html')

@app.route('/dashboard')
@login_required
def dashboard():
    # Get data based on user role
    if current_user.role == 'admin':
        stats = {
            'students': Student.query.count(),
            'teachers': Teacher.query.count(),
            'parents': Parent.query.count(),
            'classes': Class.query.count()
        }
        return render_template('admin/dashboard.html', stats=stats)
    
    elif current_user.role == 'teacher':
        teacher = Teacher.query.filter_by(user_id=current_user.id).first()
        classes = Class.query.filter_by(teacher_id=teacher.id).all()
        return render_template('teacher/dashboard.html', classes=classes)
    
    elif current_user.role == 'student':
        student = Student.query.filter_by(user_id=current_user.id).first()
        class_links = ClassStudent.query.filter_by(student_id=student.id).all()
        classes = [link.class_ for link in class_links]
        
        # Get upcoming assignments
        assignments = []
        for class_ in classes:
            class_assignments = Assignment.query.filter(
                Assignment.class_id == class_.id,
                Assignment.due_date > datetime.now()
            ).all()
            assignments.extend(class_assignments)
        
        # Sort by due date
        assignments.sort(key=lambda x: x.due_date)
        
        return render_template('student/dashboard.html', 
                              student=student, 
                              classes=classes, 
                              assignments=assignments[:5])  # Show only 5 upcoming
    
    elif current_user.role == 'parent':
        parent = Parent.query.filter_by(user_id=current_user.id).first()
        children_links = StudentParent.query.filter_by(parent_id=parent.id).all()
        children = [link.student for link in children_links]
        
        # Get children's upcoming assignments
        assignments = []
        for child in children:
            class_links = ClassStudent.query.filter_by(student_id=child.id).all()
            for link in class_links:
                class_assignments = Assignment.query.filter(
                    Assignment.class_id == link.class_id,
                    Assignment.due_date > datetime.now()
                ).all()
                assignments.extend(class_assignments)
        
        # Sort by due date
        assignments.sort(key=lambda x: x.due_date)
        
        return render_template('parent/dashboard.html', 
                              parent=parent, 
                              children=children, 
                              assignments=assignments[:5])

# Authentication routes
@app.route('/login', methods=['GET', 'POST'])
def login():
    if current_user.is_authenticated:
        return redirect(url_for('dashboard'))
    
    if request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        remember = bool(request.form.get('remember'))
        
        user = User.query.filter_by(username=username).first()
        
        if user and user.check_password(password) and user.is_active:
            login_user(user, remember=remember)
            user.last_login = datetime.utcnow()
            db.session.commit()
            
            next_page = request.args.get('next')
            return redirect(next_page or url_for('dashboard'))
        else:
            flash('Invalid username or password', 'danger')
    
    return render_template('auth/login.html')

@app.route('/logout')
@login_required
def logout():
    logout_user()
    flash('You have been logged out.', 'info')
    return redirect(url_for('index'))

# Student routes
@app.route('/student/grades')
@login_required
@role_required(['student'])
def student_grades():
    student = Student.query.filter_by(user_id=current_user.id).first()
    class_links = ClassStudent.query.filter_by(student_id=student.id).all()
    
    grades_data = []
    for link in class_links:
        class_grades = Grade.query.filter_by(
            student_id=student.id,
            class_id=link.class_id
        ).all()
        
        if class_grades:
            class_avg = sum(grade.score for grade in class_grades) / len(class_grades)
            grades_data.append({
                'class': link.class_,
                'grades': class_grades,
                'average': class_avg
            })
    
    return render_template('student/grades.html', grades_data=grades_data)

# Teacher routes
@app.route('/teacher/classes')
@login_required
@role_required(['teacher'])
def teacher_classes():
    teacher = Teacher.query.filter_by(user_id=current_user.id).first()
    classes = Class.query.filter_by(teacher_id=teacher.id).all()
    return render_template('teacher/classes.html', classes=classes)

@app.route('/teacher/class/<int:class_id>')
@login_required
@role_required(['teacher'])
def teacher_class_detail(class_id):
    class_ = Class.query.get_or_404(class_id)
    students = ClassStudent.query.filter_by(class_id=class_id).all()
    assignments = Assignment.query.filter_by(class_id=class_id).all()
    
    return render_template('teacher/class_detail.html', 
                          class_=class_, 
                          students=students, 
                          assignments=assignments)

# Parent routes
@app.route('/parent/children')
@login_required
@role_required(['parent'])
def parent_children():
    parent = Parent.query.filter_by(user_id=current_user.id).first()
    children_links = StudentParent.query.filter_by(parent_id=parent.id).all()
    children = [link.student for link in children_links]
    
    return render_template('parent/children.html', children=children)

@app.route('/parent/child/<int:student_id>')
@login_required
@role_required(['parent'])
def parent_child_detail(student_id):
    parent = Parent.query.filter_by(user_id=current_user.id).first()
    child_link = StudentParent.query.filter_by(
        parent_id=parent.id, 
        student_id=student_id
    ).first_or_404()
    
    child = child_link.student
    class_links = ClassStudent.query.filter_by(student_id=child.id).all()
    
    # Get grades
    grades_data = []
    for link in class_links:
        class_grades = Grade.query.filter_by(
            student_id=child.id,
            class_id=link.class_id
        ).all()
        
        if class_grades:
            class_avg = sum(grade.score for grade in class_grades) / len(class_grades)
            grades_data.append({
                'class': link.class_,
                'grades': class_grades,
                'average': class_avg
            })
    
    # Get attendance
    attendance = Attendance.query.filter_by(student_id=child.id).order_by(
        Attendance.date.desc()
    ).limit(30).all()
    
    return render_template('parent/child_detail.html', 
                          child=child, 
                          grades_data=grades_data, 
                          attendance=attendance)

# Admin routes
@app.route('/admin/users')
@login_required
@role_required(['admin'])
def admin_users():
    users = User.query.all()
    return render_template('admin/users.html', users=users)

@app.route('/admin/create_user', methods=['GET', 'POST'])
@login_required
@role_required(['admin'])
def admin_create_user():
    if request.method == 'POST':
        try:
            username = request.form.get('username')
            email = request.form.get('email')
            password = request.form.get('password')
            role = request.form.get('role')
            first_name = request.form.get('first_name')
            last_name = request.form.get('last_name')
            
            # Create user
            user = User(
                username=username,
                email=email,
                role=role,
                first_name=first_name,
                last_name=last_name
            )
            user.set_password(password)
            
            db.session.add(user)
            db.session.flush()  # Get the user ID without committing
            
            # Create profile based on role
            if role == 'teacher':
                employee_id = request.form.get('employee_id')
                department = request.form.get('department')
                
                teacher = Teacher(
                    user_id=user.id,
                    employee_id=employee_id,
                    department=department
                )
                db.session.add(teacher)
            
            elif role == 'student':
                student_id = request.form.get('student_id')
                date_of_birth = datetime.strptime(request.form.get('date_of_birth'), '%Y-%m-%d')
                grade_level = request.form.get('grade_level')
                
                student = Student(
                    user_id=user.id,
                    student_id=student_id,
                    date_of_birth=date_of_birth,
                    grade_level=grade_level,
                    enrollment_date=datetime.utcnow()
                )
                db.session.add(student)
            
            elif role == 'parent':
                phone = request.form.get('phone')
                occupation = request.form.get('occupation')
                
                parent = Parent(
                    user_id=user.id,
                    phone=phone,
                    occupation=occupation
                )
                db.session.add(parent)
            
            db.session.commit()
            flash('User created successfully', 'success')
            return redirect(url_for('admin_users'))
        
        except IntegrityError:
            db.session.rollback()
            flash('Username or email already exists', 'danger')
        except Exception as e:
            db.session.rollback()
            flash(f'Error creating user: {str(e)}', 'danger')
    
    return render_template('admin/create_user.html')

# API routes for AJAX calls
@app.route('/api/attendance/<int:class_id>/<date>')
@login_required
@role_required(['teacher'])
def api_attendance(class_id, date):
    attendance_date = datetime.strptime(date, '%Y-%m-%d').date()
    class_students = ClassStudent.query.filter_by(class_id=class_id).all()
    
    attendance_data = []
    for cs in class_students:
        attendance = Attendance.query.filter_by(
            student_id=cs.student_id,
            class_id=class_id,
            date=attendance_date
        ).first()
        
        attendance_data.append({
            'student_id': cs.student_id,
            'student_name': f"{cs.student.user.first_name} {cs.student.user.last_name}",
            'status': attendance.status if attendance else 'absent'
        })
    
    return jsonify(attendance_data)

@app.route('/api/attendance/<int:class_id>/<date>', methods=['POST'])
@login_required
@role_required(['teacher'])
def api_update_attendance(class_id, date):
    attendance_date = datetime.strptime(date, '%Y-%m-%d').date()
    data = request.get_json()
    
    for record in data:
        attendance = Attendance.query.filter_by(
            student_id=record['student_id'],
            class_id=class_id,
            date=attendance_date
        ).first()
        
        if attendance:
            attendance.status = record['status']
        else:
            attendance = Attendance(
                student_id=record['student_id'],
                class_id=class_id,
                date=attendance_date,
                status=record['status']
            )
            db.session.add(attendance)
    
    db.session.commit()
    return jsonify({'success': True})

# Error handlers
@app.errorhandler(404)
def not_found_error(error):
    return render_template('errors/404.html'), 404

@app.errorhandler(500)
def internal_error(error):
    db.session.rollback()
    return render_template('errors/500.html'), 500

# Initialize database
def init_db():
    with app.app_context():
        db.create_all()
        
        # Create admin user if it doesn't exist
        if not User.query.filter_by(username='admin').first():
            admin = User(
                username='admin',
                email='admin@schoolapp.com',
                role='admin',
                first_name='Admin',
                last_name='User'
            )
            admin.set_password('admin123')
            db.session.add(admin)
            db.session.commit()
            print("Admin user created: username=admin, password=admin123")

if __name__ == '__main__':
    init_db()
    app.run(debug=True)
