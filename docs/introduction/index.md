from flask import Flask, render_template, request, redirect, session
from flask_sqlalchemy import SQLAlchemy
import random
import hashlib

app = Flask(__name__)
app.secret_key = 'your_secret_key'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///platform.db'
db = SQLAlchemy(app)

# 数据库模型
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True)
    password = db.Column(db.String(120))
    coins = db.Column(db.Integer, default=100)

class Card(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100))
    rarity = db.Column(db.String(50))
    owner_id = db.Column(db.Integer, db.ForeignKey('user.id'))

class Friendship(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer)
    friend_id = db.Column(db.Integer)

# 卡池配置
CARD_POOL = {
    "Common": ["战士", "法师", "盗贼"],
    "Rare": ["龙骑士", "大魔导师", "暗影刺客"],
    "Epic": ["圣光天使", "地狱魔王"],
    "Legendary": ["创世神"]
}

# 路由配置
@app.route('/')
def index():
    if 'user_id' in session:
        return redirect('/dashboard')
    return render_template('index.html')

@app.route('/login', methods=['POST'])
def login():
    username = request.form['username']
    password = hashlib.sha256(request.form['password'].encode()).hexdigest()
    user = User.query.filter_by(username=username, password=password).first()
    if user:
        session['user_id'] = user.id
    return redirect('/')

@app.route('/register', methods=['POST'])
def register():
    username = request.form['username']
    password = hashlib.sha256(request.form['password'].encode()).hexdigest()
    new_user = User(username=username, password=password)
    db.session.add(new_user)
    db.session.commit()
    return redirect('/')

@app.route('/dashboard')
def dashboard():
    if 'user_id' not in session:
        return redirect('/')
    user = User.query.get(session['user_id'])
    return render_template('dashboard.html', user=user)

@app.route('/gacha')
def gacha():
    if 'user_id' not in session:
        return redirect('/')
    
    user = User.query.get(session['user_id'])
    if user.coins < 10:
        return "金币不足"
    
    # 抽卡逻辑
    rarity = random.choices(
        ["Common", "Rare", "Epic", "Legendary"],
        weights=[60, 30, 8, 2]
    )[0]
    
    new_card = Card(
        name=random.choice(CARD_POOL[rarity]),
        rarity=rarity,
        owner_id=user.id
    )
    
    user.coins -= 10
    db.session.add(new_card)
    db.session.commit()
    return redirect('/cards')

@app.route('/cards')
def cards():
    user = User.query.get(session['user_id'])
    cards = Card.query.filter_by(owner_id=user.id).all()
    return render_template('cards.html', cards=cards)

@app.route('/trade', methods=['POST'])
def trade():
    # 实现换卡逻辑
    pass

@app.route('/friends')
def friends():
    # 实现好友系统
    pass

@app.route('/profile')
def profile():
    # 用户个人中心
    pass

@app.route('/logout')
def logout():
    session.pop('user_id', None)
    return redirect('/')

if __name__ == '__main__':
    with app.app_context():
        db.create_all()
    app.run(debug=True)
