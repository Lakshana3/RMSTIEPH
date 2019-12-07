#!/usr/bin/python

from flask import Flask, render_template, request, redirect, url_for, flash, session
from dbconnect import connection
from pulse import bpm
from thermometer import read_temp
from wtforms import Form, StringField, PasswordField, validators, BooleanField, RadioField
from passlib.hash import sha256_crypt
from functools import wraps
from MySQLdb import escape_string as thwart
import serial, datetime, time, glob, os
import gc

dateString = '%Y-%m-%d %H-%M-%S'

app = Flask(__name__)
app.secret_key='omsakthi3'

################################################################################################################################################################

@app.route('/') # Main page
def homepage():
    return render_template("main.html")

########################################################################################################################################################

def login_required(f): # login required
    @wraps(f)
    def wrap(*args, **kwargs):
        if 'logged_in' in session:
            return f(*args, **kwargs)
        else:
            flash("You need to login first")
            return redirect(url_for('login'))
    return wrap

########################################################################################################################################

@app.route("/logout/") # logout
@login_required
def logout():
    session.clear()
    flash("You have been logged out!")
    gc.collect()
    return redirect(url_for('homepage'))

#########################################################################################################################################################################################################################
# PATIENT
#######################################################################################################################################################################################################################################

@app.route('/dashboard_p/',methods=['GET','POST']) # patient's dashboard
@login_required
def dashboard_p():
    try:
        curs,conn = connection()
        if request.method == "POST":
                for i in range (1,60):
                    bpm = bpm
                    temp = read_temp()
                    dt = datetime.datetime.now().strftime(dateString)
                    curs.execute("INSERT into {}(DateTime, Temperature, PulseRate) VALUES (%s,%s,%s)" . format(session['username']), ( dt, temp, bpm))
                    conn.commit()
                    time.sleep(1)
        curs.execute("SELECT DateTime,Temperature,PulseRate from %s" % (session['username']) )
        data = curs.fetchall()
        curs.close()
        conn.close()
        gc.collect()
        return render_template("dashboard_p.html", data = data)

    except Exception as e:
        return (str(e))
       
#############################################################################################################################################################################################################

@app.route('/profile_p/',methods=['GET','POST']) # patient's profile
def profile_p():
    try:
        curs,conn = connection()
        username = session['username']
        if request.method == "POST":
            flash("Your profile is updated!")
            email = request.form['email1']
            curs.execute("UPDATE patients SET email = (%s) WHERE username = (%s)",( thwart(email), thwart(username)))
            conn.commit()
            age = request.form['age1']
            curs.execute("UPDATE patients SET age = (%s) WHERE username = (%s)",( thwart(age), thwart(username)))
            conn.commit()
            allergies = request.form['allergies1']
            curs.execute("UPDATE patients SET allergies = (%s)  WHERE username = (%s)",( thwart(allergies), thwart(username)))
            conn.commit()
            
        email = curs.execute("SELECT * FROM patients WHERE username = (%s)",
                                     (thwart(session['username']),))
        email = curs.fetchone()[3]
            
        age = curs.execute("SELECT * FROM patients WHERE username = (%s)",
                                     (thwart(session['username']),))
        age = curs.fetchone()[4]
            
        gender = curs.execute("SELECT * FROM patients WHERE username = (%s)",
                                     (thwart(session['username']),))
        gender = curs.fetchone()[5]
            
        allergies = curs.execute("SELECT * FROM patients WHERE username = (%s)",
                                     (thwart(session['username']),))
        allergies = curs.fetchone()[7]
            
        conn.close()
        curs.close()
        gc.collect()
        return render_template("profile_p.html",username=session['username'],email = email,age = age,gender = gender,allergies = allergies)

    except Exception as e:
        return(str(e))

##########################################################################################################################################################################
    
@app.route('/messages_p/') # Patient's message page
def messages_p():
    try:
        curs,conn = connection()
        query = "SELECT username FROM doctors"
        curs.execute(query)
        data = curs.fetchall()
        conn.close()
        return render_template("messages_p.html", data = data) # renders names of doctors

    except Exception as e:
        return (str(e))

######################################################################################################################################################################################

@app.route('/messagesp/<username>', methods=['GET','POST']) # doctor's messages in patients account
def messagep(username):
    try:
        name = username # doctor name
        username = session['username'] # patient name -> username
        curs,conn = connection()
        if request.method == "POST":
            query = request.form['query']
            dt = datetime.datetime.now().strftime(dateString)
            curs.execute("INSERT INTO {}(DateTime,doctorname,messages) VALUES ( %s, %s, %s)".format(username),(dt, name, query))
            conn.commit()
        curs.execute("SELECT DateTime,messages FROM {} WHERE username = (%s)".format(name),(username,))
        data = curs.fetchall()
        conn.close()
        return render_template("messagep.html", data=data)

    except Exception as e:
        return(str(e))

#####################################################################################################################################################################################################
# DOCTOR
############################################################################################################################################################################################################

@app.route('/dashboard_m/') # doctor's dashboard
@login_required
def dashboard_m():
    try:
        curs,conn = connection()
        query = "SELECT username FROM patients"
        curs.execute(query)
        data = curs.fetchall()
        conn.close()
        return render_template("dashboard_m.html", data = data)

    except Exception as e:
        return (str(e))

####################################################################################################################################################

@app.route('/patient/<username>',methods=['GET','POST']) # redirects from doctor's dashboard
def patient(username):
    try:
        curs,conn = connection()
        name = session['username']
        if request.method == "POST":
            comment = request.form['comment']
            dt = datetime.datetime.now().strftime(dateString)
            curs.execute("INSERT INTO {} VALUES ( %s, %s, %s)" . format(name),(dt, username, comment))
            conn.commit()
        curs.execute("SELECT DateTime,Temperature,PulseRate from {} " . format(username))
        data = curs.fetchall()
        conn.close()
        curs.close()
        gc.collect()
        return render_template("patient.html", data = data)

    except Exception as e:
        return (str(e))

###################################################################################################################################################################################################################################################

@app.route('/profile_m/',methods=['GET','POST']) # doctor's profile page
def profile_m():
    try:
        curs,conn = connection()
        username = session['username']
        if request.method == "POST":
            flash("Your profile is updated!")
            email = request.form['email1']
            curs.execute("UPDATE doctors SET email = (%s) WHERE username = (%s)",( thwart(email), username))
            conn.commit()
            age = request.form['age1']
            curs.execute("UPDATE doctors SET age = (%s) WHERE username = (%s)",( thwart(age), username))
            conn.commit()
            qual = request.form['qual1']
            curs.execute("UPDATE doctors SET qual = (%s) WHERE username = (%s)",( thwart(qual), username))
            conn.commit()
            cwas = request.form['cwas1']
            curs.execute("UPDATE doctors SET cwas = (%s) WHERE username = (%s)",( thwart(cwas), username))
            conn.commit()
            cwat = request.form['cwat1']
            curs.execute("UPDATE doctors SET cwat = (%s) WHERE username = (%s)",( thwart(cwat), username))
            conn.commit()  
        
        data = curs.execute("SELECT * FROM doctors WHERE username = (%s)",
                                     (thwart(session['username']),))
        email = curs.fetchone()[3]
            
        data = curs.execute("SELECT * FROM doctors WHERE username = (%s)",
                                     (thwart(session['username']),))
        age = curs.fetchone()[4]
            
        data = curs.execute("SELECT * FROM doctors WHERE username = (%s)",
                                     (thwart(session['username']),))
        gender = curs.fetchone()[5]
            
        data = curs.execute("SELECT * FROM doctors WHERE username = (%s)",
                                     (thwart(session['username']),))
        qual = curs.fetchone()[7]
            
        data = curs.execute("SELECT * FROM doctors WHERE username = (%s)",
                                     (thwart(session['username']),))
        cwas = curs.fetchone()[8]
            
        data = curs.execute("SELECT * FROM doctors WHERE username = (%s)",
                                     (thwart(session['username']),))
        cwat = curs.fetchone()[9]
            
        conn.close()
        curs.close()
        gc.collect()
        return render_template("profile_m.html",username=session['username'],email = email,age = age,gender = gender,qual = qual, cwas = cwas, cwat = cwat)

    except Exception as e:
        return(str(e))

###############################################################################################################################
        
@app.route('/messages_m/') # doctor's message page
def messages_m():
    try:
        curs,conn = connection()
        query = "SELECT username FROM patients"
        curs.execute(query)
        data = curs.fetchall()
        conn.close()
        return render_template("messages_m.html", data = data)

    except Exception as e:
        return (str(e))

##############################################################################################################################

@app.route('/messagesm/<username>',methods=['POST','GET']) # patient's queries
def messagem(username):
    try:
        curs,conn = connection()
        name = username # patient's name
        username = session['username'] # doctor's name -> username 
        curs.execute("SELECT DateTime,messages FROM {} WHERE doctorname = (%s)".format(name),(username,))
        data = curs.fetchall()
        conn.close()
        return render_template("messagem.html", data=data)

    except Exception as e:
        return (str(e))

###############################################################################################################################################

@app.route('/profilem/<username>',methods=['GET'])
def profilem(username):
    try:
        curs,conn = connection()
        username = username
        data = curs.execute("SELECT * FROM doctors WHERE username = (%s)",
                                     (thwart(username),))
        email = curs.fetchone()[3]
            
        data = curs.execute("SELECT * FROM doctors WHERE username = (%s)",
                                     (thwart(username),))
        age = curs.fetchone()[4]
            
        data = curs.execute("SELECT * FROM doctors WHERE username = (%s)",
                                     (thwart(username),))
        gender = curs.fetchone()[5]
            
        data = curs.execute("SELECT * FROM doctors WHERE username = (%s)",
                                     (thwart(username),))
        qual = curs.fetchone()[7]
            
        data = curs.execute("SELECT * FROM doctors WHERE username = (%s)",
                                     (thwart(username),))
        cwas = curs.fetchone()[8]
            
        data = curs.execute("SELECT * FROM doctors WHERE username = (%s)",
                                     (thwart(username),))
        cwat = curs.fetchone()[9]
        
        return render_template("profilem.html", username=username,email = email,age = age,gender = gender,qual = qual, cwas = cwas, cwat = cwat)

    except Exception as e:
        return (str(e))
    
###############################################################################################################################################

@app.route('/profilep/<username>',methods=['POST','GET'])
def profilep(username):
    try:
        curs,conn = connection()
        username = username
        email = curs.execute("SELECT * FROM patients WHERE username = (%s)",
                                     (thwart(username),))
        email = curs.fetchone()[3]
            
        age = curs.execute("SELECT * FROM patients WHERE username = (%s)",
                                     (thwart(username),))
        age = curs.fetchone()[4]
            
        gender = curs.execute("SELECT * FROM patients WHERE username = (%s)",
                                     (thwart(username),))
        gender = curs.fetchone()[5]
            
        allergies = curs.execute("SELECT * FROM patients WHERE username = (%s)",
                                     (thwart(username),))
        allergies = curs.fetchone()[7]
        conn.close()
        return render_template("profilep.html", username=username,email = email,age = age,gender = gender,allergies = allergies)

    except Exception as e:
        return (str(e))
    
###############################################################################################################################
###############################################################################################################################

@app.route('/login/',methods=['GET','POST']) # login page
def login():
    flash('Omsakthi')
    error = ''
    try:
        curs, conn = connection()
        if request.method == "POST":
            if request.form['tou'] == 'm':
                data = curs.execute("SELECT * FROM doctors WHERE username = (%s)",
                                 (thwart(request.form['username']),))                
                data = curs.fetchone()[2]              
                if sha256_crypt.verify(request.form['password'], data):
                    session['logged_in'] = True
                    session['username'] = request.form['username']
                    flash("You are now logged in")
                    return redirect(url_for("dashboard_m"))                   
                else:
                    error = "Invalid credentials, try again."
            else:
                data = curs.execute("SELECT * FROM patients WHERE username = (%s)",
                                 (thwart(request.form['username']),))               
                data = curs.fetchone()[2]              
                if sha256_crypt.verify(request.form['password'], data):
                    session['logged_in'] = True
                    session['username'] = request.form['username']
                    flash("You are now logged in")
                    return redirect(url_for("dashboard_p"))                    
                else:
                    error = "Invalid credentials, try again."
        gc.collect()
        return render_template("login.html", error=error)

    except Exception as e:
        #flash(e)
        error = "Invalid credentials, try again."
        return render_template("login.html", error = error)

##################################################################################################################

class RegistrationForm(Form):
    username = StringField('Username', [validators.Length(min=4, max=20)])
    email = StringField('Email Address', [validators.Length(min=6, max=50)])
    password = PasswordField('New Password', [
        validators.DataRequired(),
        validators.EqualTo('confirm', message='Passwords must match')
    ])
    confirm = PasswordField('Repeat Password')
    age = StringField('Age')
    gender = RadioField('Gender', choices = [('M','Male'),('F','Female')])
    tou = RadioField('Type of user', choices = [('M','Medical_practitioner'),('P','Patient')])
    accept_tos = BooleanField('I accept the Terms of Service and Privacy Notice', [validators.Required()])

###################################################################################################################

@app.route('/register/',methods=['GET','POST']) # register page
def signup():
    try:
        form = RegistrationForm(request.form)
        if request.method == "POST" and form.validate():
            username  = form.username.data
            email = form.email.data
            password = sha256_crypt.encrypt((str(form.password.data)))
            age = form.age.data
            gender = form.gender.data
            tou = form.tou.data
            curs, conn = connection()
            x = curs.execute("SELECT * FROM doctors WHERE username = (%s)",
                          (thwart(username),))
            y = curs.execute("SELECT * FROM patients WHERE username = (%s)",
                          (thwart(username),))
            if int(x) > 0 or int(y) > 0:
                flash("That username is already taken, please choose another")
                return render_template('register.html', form=form)
            else:
                if tou == 'P':
                    allergies= 'NIL'
                    curs.execute("INSERT INTO patients  ( username, password, email, age, gender, allergies) VALUES (%s, %s, %s, %s, %s, %s)",
                              ( thwart(username), thwart(password), thwart(email), thwart(age), thwart(gender), thwart(allergies)))                    
                    conn.commit()
                    flash("Thanks for registering!")
                    #curs.close()
                    #conn.close()
                    #gc.collect()
                    session['logged_in'] = True
                    session['username'] = username
                    
                    curs,conn = connection()
                    curs.execute("CREATE TABLE {} (DateTime datetime, Temperature VARCHAR(10), PulseRate VARCHAR(10), doctorname VARCHAR(50), messages VARCHAR(32500))" .format(session['username']))
                    conn.commit()
                    curs.close()
                    conn.close()
                    gc.collect()
                    return redirect(url_for('dashboard_p'))
                else:
                    qual='NIL'
                    cwat='NIL'
                    cwas='NIL'
                    curs.execute("INSERT INTO doctors ( username, password, email, age, gender, qual, cwas, cwat) VALUES (%s, %s, %s, %s, %s, %s, %s, %s)",
                              ( thwart(username), thwart(password), thwart(email), thwart(age), thwart(gender), thwart(qual), thwart(cwas), thwart(cwat)))                    
                    conn.commit()
                    flash("Thanks for registering!")
                    #curs.close()
                    #conn.close()
                    #gc.collect()
                    session['logged_in'] = True
                    session['username'] = username
                    session['email'] = email
                    
                    curs,conn = connection()
                    curs.execute("CREATE TABLE {} (DateTime datetime , username VARCHAR(50), messages VARCHAR(32500))" . format(session['username']))
                    conn.commit()
                    curs.close()
                    conn.close()
                    gc.collect()                   
                    return redirect(url_for('dashboard_m')) 
        return render_template("register.html", form=form)

    except Exception as e:
        return(str(e))

#################################################################################

@app.errorhandler(404)
def page_not_found(e):
    return render_template("404.html")

#################################################################################

@app.errorhandler(405)
def method_not_allowed():
    return render_template("405.html")

########################################################################################################################################################################
########################################################################################################################################################################

if __name__ == "__main__":
    app.run(debug=True)
