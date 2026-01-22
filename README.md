import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, collection, onSnapshot, updateDoc, addDoc, deleteDoc, serverTimestamp } from 'firebase/firestore';
import { 
  Plus, Trash2, Store, MapPin, Phone, Settings, X, ShoppingBag, 
  Truck, Archive, Lock, Star, MessageSquare, Edit3, RefreshCw, 
  User, Image as ImageIcon, Clock, LayoutGrid
} from 'lucide-react';

const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'ks-closet-v16';

const CATEGORIES = [
  { id: 'all', name: 'ทั้งหมด', emoji: '🏠' },
  { id: 'clothes', name: 'เสื้อผ้า', emoji: '👕', sizes: ['M', 'L', 'XL', '2XL', '3XL'] },
  { id: 'pants', name: 'กางเกง', emoji: '👖', sizes: ['28', '30', '32', '34', '36', '38'] },
  { id: 'shoes', name: 'รองเท้า', emoji: '👟', sizes: ['36', '37', '38', '39', '40', '41', '42', '43', '44', '45'] },
  { id: 'glasses', name: 'แว่นตา', emoji: '🕶️', sizes: ['Free Size'] },
  { id: 'jewelry', name: 'เครื่องประดับ', emoji: '💍', sizes: ['Free Size'] },
  { id: 'hats', name: 'หมวก', emoji: '🧢', sizes: ['Free Size'] }
];

export default function App() {
  const [user, setUser] = useState(null);
  const [isAdminMode, setIsAdminMode] = useState(false);
  const [adminPass, setAdminPass] = useState('');
  const [showAdminLogin, setShowAdminLogin] = useState(false);
  const [view, setView] = useState('shop');
  
  const [products, setProducts] = useState([]);
  const [orders, setOrders] = useState([]);
  const [reviews, setReviews] = useState([]);
  const [activeCat, setActiveCat] = useState('all');

  const [selectedProduct, setSelectedProduct] = useState(null);
  const [selectedSize, setSelectedSize] = useState('');
  const [isBackView, setIsBackView] = useState(false);
  const [customerInfo, setCustomerInfo] = useState({ name: '', address: '', phone: '' });
  const [reviewForm, setReviewForm] = useState({ text: '', rating: 5 });

  const [isFormOpen, setIsFormOpen] = useState(false);
  const [editingId, setEditingId] = useState(null);
  const [productForm, setProductForm] = useState({ name: '', price: '', category: 'clothes', frontImg: '', backImg: '', sizeStocks: {} });

  // Sound Utility
  const playSound = (type = 'click') => {
    const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    const oscillator = audioCtx.createOscillator();
    const gainNode = audioCtx.createGain();

    oscillator.connect(gainNode);
    gainNode.connect(audioCtx.destination);

    if (type === 'click') {
      // Relaxing high-pitch chime
      oscillator.type = 'sine';
      oscillateor.frequency.setValueAtTime(880, audioCtx.currentTime);
      oscillator.frequency.exponentialRampToValueAtTime(440, audioCtx.currentTime + 0.1);
      gainNode.gain.setValueAtTime(0.1, audioCtx.currentTime);
      gainNode.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 0.1);
      oscillator.start();
      oscillator.stop(audioCtx.currentTime + 0.1);
    } else if (type === 'success') {
      // Heavenly soft chime
      oscillator.type = 'triangle';
      oscillator.frequency.setValueAtTime(523.25, audioCtx.currentTime);
      oscillator.frequency.exponentialRampToValueAtTime(1046.50, audioCtx.currentTime + 0.3);
      gainNode.gain.setValueAtTime(0.05, audioCtx.currentTime);
      gainNode.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.3);
      oscillator.start();
      oscillator.stop(audioCtx.currentTime + 0.3);
    }
  };

  useEffect(() => {
    const initAuth = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else { await signInAnonymously(auth); }
    };
    initAuth();
    onAuthStateChanged(auth, setUser);
  }, []);

  useEffect(() => {
    if (!user) return;
    const unsubProd = onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'inventory'), (snap) => {
      setProducts(snap.docs.map(d => ({ id: d.id, ...d.data() })));
    });
    const unsubOrder = onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'orders'), (snap) => {
      setOrders(snap.docs.map(d => ({ id: d.id, ...d.data() })).sort((a, b) => (b.timestamp?.seconds || 0) - (a.timestamp?.seconds || 0)));
    });
    const unsubReviews = onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'reviews'), (snap) => {
      setReviews(snap.docs.map(d => ({ id: d.id, ...d.data() })));
    });
    return () => { unsubProd(); unsubOrder(); unsubReviews(); };
  }, [user]);

  const handleAdminAuth = () => {
    playSound();
    if (adminPass === '1234') {
      setIsAdminMode(true);
      setShowAdminLogin(false);
      setAdminPass('');
      setView('admin_orders');
    } else {
      alert("รหัสผ่านไม่ถูกต้อง");
    }
  };

  const handleSaveProduct = async () => {
    playSound('success');
    const id = editingId || Date.now().toString();
    await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'inventory', id), {
      ...productForm,
      id,
      price: Number(productForm.price),
      timestamp: serverTimestamp()
    });
    setIsFormOpen(false);
    setEditingId(null);
  };

  const handleOrder = async () => {
    if (!selectedSize || !customerInfo.name || !customerInfo.phone) return alert("กรุณากรอกข้อมูล");
    playSound('success');
    
    const stock = selectedProduct.sizeStocks[selectedSize] || 0;
    const newStocks = { ...selectedProduct.sizeStocks, [selectedSize]: stock - 1 };
    await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'inventory', selectedProduct.id), { sizeStocks: newStocks });

    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'orders'), {
      ...customerInfo,
      productName: selectedProduct.name,
      productId: selectedProduct.id,
      size: selectedSize,
      price: selectedProduct.price,
      imageUrl: selectedProduct.frontImg,
      status: 'ออเดอร์ใหม่',
      timestamp: serverTimestamp(),
      customerId: user.uid
    });
    setSelectedProduct(null);
    setView('my_orders');
  };

  const handleMenuClick = (newView) => {
    playSound();
    setView(newView);
  };

  return (
    <div className="max-w-md mx-auto min-h-screen bg-[#FDFCFB] text-slate-800 font-sans pb-28">
      
      {/* HEADER */}
      <header className="bg-white/80 backdrop-blur-md border-b px-6 py-5 flex justify-between items-center sticky top-0 z-[60]">
        <div>
          <h1 className="text-2xl font-black italic tracking-tighter text-indigo-500">KS CLOSET</h1>
          <p className="text-[10px] font-bold text-slate-300 uppercase tracking-widest">
            {isAdminMode ? 'ADMIN PANEL' : 'EXCLUSIVE SHOP'}
          </p>
        </div>
        <button 
          onClick={() => { playSound(); setShowAdminLogin(true); }}
          className={`w-12 h-12 rounded-2xl flex items-center justify-center transition-all ${isAdminMode ? 'bg-indigo-500 text-white' : 'bg-slate-50 text-slate-400'}`}
        >
          {isAdminMode ? <Settings size={20} /> : <Lock size={20} />}
        </button>
      </header>

      {/* ADMIN LOGIN */}
      {showAdminLogin && (
        <div className="fixed inset-0 bg-slate-900/60 backdrop-blur-sm z-[100] flex items-center justify-center p-6">
          <div className="bg-white rounded-[40px] p-8 w-full max-w-xs shadow-2xl">
            <h3 className="text-xl font-black text-center mb-6">ผู้บริหารร้าน</h3>
            <input 
              type="password" 
              placeholder="รหัสผ่าน (1234)" 
              className="w-full p-5 bg-slate-50 rounded-2xl mb-4 text-center text-xl tracking-widest outline-none border-2 border-transparent focus:border-indigo-100"
              value={adminPass}
              onChange={(e) => setAdminPass(e.target.value)}
            />
            <div className="flex gap-2">
              <button onClick={handleAdminAuth} className="flex-1 bg-indigo-500 text-white py-4 rounded-2xl font-bold shadow-lg shadow-indigo-100">ยืนยัน</button>
              <button onClick={() => setShowAdminLogin(false)} className="flex-1 bg-slate-50 text-slate-400 py-4 rounded-2xl font-bold">ยกเลิก</button>
            </div>
            {isAdminMode && (
              <button onClick={() => { setIsAdminMode(false); setShowAdminLogin(false); setView('shop'); }} className="w-full mt-2 text-red-400 text-xs font-bold py-2">ออกจากโหมด Admin</button>
            )}
          </div>
        </div>
      )}

      {/* MAIN VIEW */}
      <main className="px-6 pt-6">
        
        {/* SHOP VIEW */}
        {!isAdminMode && view === 'shop' && (
          <>
            <div className="flex gap-2 overflow-x-auto no-scrollbar pb-6">
              {CATEGORIES.map(cat => (
                <button
                  key={cat.id}
                  onClick={() => { playSound(); setActiveCat(cat.id); }}
                  className={`flex-none px-6 py-3 rounded-2xl text-[11px] font-black transition-all ${activeCat === cat.id ? 'bg-slate-900 text-white shadow-xl' : 'bg-white text-slate-400 shadow-sm border border-slate-50'}`}
                >
                  {cat.emoji} {cat.name}
                </button>
              ))}
            </div>

            <div className="grid grid-cols-2 gap-5">
              {products.filter(p => activeCat === 'all' || p.category === activeCat).map(p => {
                const totalStock = Object.values(p.sizeStocks || {}).reduce((a, b) => a + b, 0);
                return (
                  <div key={p.id} onClick={() => { playSound(); setSelectedProduct(p); }} className="active:scale-95 transition-all">
                    <div className="aspect-[3/4] rounded-[35px] overflow-hidden bg-white shadow-sm border border-slate-100 relative mb-3">
                      <img src={p.frontImg} className="w-full h-full object-cover" />
                      {totalStock <= 0 && (
                        <div className="absolute inset-0 bg-white/60 backdrop-blur-[2px] flex items-center justify-center">
                          <span className="bg-red-500 text-white text-[9px] font-black px-3 py-1.5 rounded-full italic tracking-tighter">SOLD OUT</span>
                        </div>
                      )}
                    </div>
                    <div className="px-1 text-center">
                      <h4 className="font-bold text-[11px] text-slate-600 truncate">{p.name}</h4>
                      <p className="text-indigo-500 font-black text-lg italic">฿{p.price.toLocaleString()}</p>
                    </div>
                  </div>
                );
              })}
            </div>
          </>
        )}

        {/* CUSTOMER ORDERS */}
        {!isAdminMode && view === 'my_orders' && (
          <div className="space-y-4">
            <h2 className="text-xl font-black mb-6 italic">ORDER HISTORY</h2>
            {orders.filter(o => o.customerId === user?.uid).map(o => (
              <div key={o.id} className="bg-white p-5 rounded-[30px] border border-slate-50 flex gap-4 shadow-sm items-center">
                <img src={o.imageUrl} className="w-16 h-16 rounded-2xl object-cover" />
                <div className="flex-1">
                  <div className="flex justify-between">
                    <h4 className="font-bold text-xs">{o.productName}</h4>
                    <span className={`text-[8px] font-black px-2 py-1 rounded-full ${o.status === 'ส่งแล้ว' ? 'bg-green-100 text-green-600' : 'bg-orange-100 text-orange-600'}`}>{o.status}</span>
                  </div>
                  <p className="text-[10px] font-bold text-slate-400 mt-1">ไซส์: {o.size} | ฿{o.price.toLocaleString()}</p>
                </div>
              </div>
            ))}
          </div>
        )}

        {/* ADMIN ORDERS */}
        {isAdminMode && view === 'admin_orders' && (
          <div className="space-y-4">
            <h2 className="text-xl font-black italic mb-6">MANAGE ORDERS</h2>
            {orders.map(o => (
              <div key={o.id} className="bg-white p-6 rounded-[35px] shadow-sm border border-slate-50 space-y-4">
                <div className="flex gap-4">
                  <img src={o.imageUrl} className="w-20 h-20 rounded-3xl object-cover" />
                  <div className="flex-1">
                    <span className={`text-[9px] font-black px-2 py-1 rounded-full ${o.status === 'ส่งแล้ว' ? 'bg-green-100 text-green-600' : 'bg-orange-100 text-orange-600'}`}>{o.status}</span>
                    <h4 className="font-bold text-sm mt-2">{o.productName}</h4>
                    <p className="text-xs font-bold text-slate-400">ไซส์: {o.size} | ฿{o.price.toLocaleString()}</p>
                  </div>
                </div>
                <div className="bg-slate-50 p-4 rounded-2xl text-[10px] font-bold space-y-1">
                  <div className="flex gap-2"><User size={12}/> {o.name}</div>
                  <div className="flex gap-2"><Phone size={12}/> {o.phone}</div>
                  <div className="flex gap-2"><MapPin size={12}/> {o.address}</div>
                </div>
                <div className="flex gap-2">
                  {o.status === 'ออเดอร์ใหม่' && <button onClick={() => { playSound(); updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'orders', o.id), {status: 'กำลังจัดส่ง'}); }} className="flex-1 bg-slate-900 text-white py-3.5 rounded-2xl font-black text-[10px] uppercase">รับออเดอร์</button>}
                  {o.status === 'กำลังจัดส่ง' && <button onClick={() => { playSound(); updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'orders', o.id), {status: 'ส่งแล้ว'}); }} className="flex-1 bg-indigo-500 text-white py-3.5 rounded-2xl font-black text-[10px] uppercase">ส่งสำเร็จ</button>}
                  <button onClick={() => { playSound(); deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'orders', o.id)); }} className="w-12 bg-red-50 text-red-400 rounded-2xl flex items-center justify-center"><Trash2 size={16}/></button>
                </div>
              </div>
            ))}
          </div>
        )}

        {/* ADMIN INVENTORY */}
        {isAdminMode && view === 'admin_inventory' && (
          <div className="space-y-6">
            <div className="flex justify-between items-center">
              <h2 className="text-xl font-black italic">INVENTORY</h2>
              <button onClick={() => { playSound(); setEditingId(null); setProductForm({name:'', price:'', category:'clothes', frontImg:'', backImg:'', sizeStocks:{}}); setIsFormOpen(true); }} className="bg-indigo-500 text-white p-3.5 rounded-2xl"><Plus/></button>
            </div>

            {isFormOpen && (
              <div className="bg-white p-8 rounded-[40px] shadow-2xl space-y-4">
                <div className="flex justify-between items-center"><h3 className="font-black">รายละเอียดสินค้า</h3><button onClick={() => setIsFormOpen(false)}><X/></button></div>
                <div className="grid grid-cols-2 gap-4">
                  <div>
                    <p className="text-[10px] font-black text-slate-300 mb-1">รูปด้านหน้า</p>
                    <input className="w-full p-3 bg-slate-50 rounded-xl text-[10px]" placeholder="URL..." value={productForm.frontImg} onChange={e => setProductForm({...productForm, frontImg: e.target.value})} />
                  </div>
                  <div>
                    <p className="text-[10px] font-black text-slate-300 mb-1">รูปด้านหลัง</p>
                    <input className="w-full p-3 bg-slate-50 rounded-xl text-[10px]" placeholder="URL..." value={productForm.backImg} onChange={e => setProductForm({...productForm, backImg: e.target.value})} />
                  </div>
                </div>
                <input placeholder="ชื่อสินค้า..." className="w-full p-4 bg-slate-50 rounded-2xl font-bold" value={productForm.name} onChange={e => setProductForm({...productForm, name: e.target.value})} />
                <div className="flex gap-2">
                  <input placeholder="ราคา..." type="number" className="flex-1 p-4 bg-slate-50 rounded-2xl font-bold" value={productForm.price} onChange={e => setProductForm({...productForm, price: e.target.value})} />
                  <select className="flex-1 p-4 bg-slate-50 rounded-2xl font-bold outline-none" value={productForm.category} onChange={e => setProductForm({...productForm, category: e.target.value, sizeStocks: {}})}>
                    {CATEGORIES.filter(c => c.id !== 'all').map(c => <option key={c.id} value={c.id}>{c.name}</option>)}
                  </select>
                </div>
                <div className="pt-2">
                   <p className="text-[10px] font-black text-slate-300 mb-2">สต็อกแยกไซส์</p>
                   <div className="grid grid-cols-4 gap-2">
                      {CATEGORIES.find(c => c.id === productForm.category)?.sizes.map(s => (
                        <div key={s} className="bg-slate-50 p-2 rounded-xl text-center">
                          <span className="text-[9px] font-bold block mb-1">{s}</span>
                          <input type="number" className="w-full bg-white text-center rounded py-1 text-xs font-bold" value={productForm.sizeStocks[s] || 0} onChange={e => setProductForm({...productForm, sizeStocks: {...productForm.sizeStocks, [s]: Number(e.target.value)}})} />
                        </div>
                      ))}
                   </div>
                </div>
                <button onClick={handleSaveProduct} className="w-full bg-indigo-500 text-white py-5 rounded-3xl font-black">บันทึกสินค้า</button>
              </div>
            )}

            <div className="space-y-3">
              {products.map(p => (
                <div key={p.id} className="bg-white p-5 rounded-[30px] shadow-sm flex items-center gap-4">
                  <img src={p.frontImg} className="w-16 h-16 rounded-2xl object-cover" />
                  <div className="flex-1">
                    <h4 className="font-bold text-xs truncate">{p.name}</h4>
                    <p className="text-indigo-500 font-black text-xs">฿{p.price.toLocaleString()}</p>
                  </div>
                  <div className="flex gap-2">
                    <button onClick={() => { playSound(); setEditingId(p.id); setProductForm(p); setIsFormOpen(true); }} className="p-2.5 bg-slate-50 text-slate-400 rounded-xl"><Edit3 size={16}/></button>
                    <button onClick={() => { playSound(); deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'inventory', p.id)); }} className="p-2.5 bg-red-50 text-red-400 rounded-xl"><Trash2 size={16}/></button>
                  </div>
                </div>
              ))}
            </div>
          </div>
        )}
      </main>

      {/* PRODUCT DETAIL */}
      {selectedProduct && (
        <div className="fixed inset-0 z-[110] flex items-end justify-center bg-slate-900/40 backdrop-blur-md p-4">
          <div className="bg-white w-full max-w-md rounded-[50px] p-8 h-[92vh] overflow-y-auto no-scrollbar relative shadow-2xl">
            <button onClick={() => { playSound(); setSelectedProduct(null); setIsBackView(false); }} className="sticky top-0 float-right z-10 w-10 h-10 bg-slate-50 rounded-full flex items-center justify-center text-slate-300"><X/></button>
            
            <div className="relative mb-8 mt-2">
              <div className="aspect-[3/4] rounded-[40px] overflow-hidden shadow-2xl bg-slate-100">
                <img src={isBackView && selectedProduct.backImg ? selectedProduct.backImg : selectedProduct.frontImg} className="w-full h-full object-cover transition-all" />
              </div>
              {selectedProduct.backImg && (
                <button onClick={() => { playSound(); setIsBackView(!isBackView); }} className="absolute bottom-6 right-6 bg-white/90 backdrop-blur px-5 py-3 rounded-2xl shadow-lg flex items-center gap-2 font-black text-[10px]">
                  <RefreshCw size={14} className={isBackView ? 'rotate-180 transition-all text-indigo-500' : 'text-slate-300'}/> สลับรูปภาพ
                </button>
              )}
            </div>

            <div className="space-y-6">
              <div>
                <h2 className="text-2xl font-black italic">{selectedProduct.name}</h2>
                <p className="text-indigo-500 text-4xl font-black mt-2">฿{selectedProduct.price.toLocaleString()}</p>
              </div>

              <div className="space-y-3">
                <p className="text-[10px] font-black text-slate-300 uppercase tracking-widest">เลือกขนาด</p>
                <div className="flex flex-wrap gap-2">
                  {CATEGORIES.find(c => c.id === selectedProduct.category)?.sizes.map(s => {
                    const stock = selectedProduct.sizeStocks[s] || 0;
                    return (
                      <button 
                        key={s} 
                        disabled={stock <= 0}
                        onClick={() => { playSound(); setSelectedSize(s); }}
                        className={`px-6 py-4 rounded-2xl font-black text-sm border-2 transition-all ${selectedSize === s ? 'bg-slate-900 text-white border-slate-900 shadow-xl' : (stock > 0 ? 'bg-white border-slate-100 text-slate-400' : 'bg-slate-50 border-transparent text-slate-200 opacity-60')}`}
                      >
                        {s} {stock <= 0 && <span className="text-[8px] block">SOLD OUT</span>}
                      </button>
                    );
                  })}
                </div>
              </div>

              <div className="space-y-3 pt-6 border-t border-slate-50">
                <p className="text-[10px] font-black text-slate-300 uppercase tracking-widest">ข้อมูลส่งของ</p>
                <input placeholder="ชื่อของคุณ..." className="w-full p-4 bg-slate-50 rounded-2xl font-bold" value={customerInfo.name} onChange={e => setCustomerInfo({...customerInfo, name: e.target.value})} />
                <input placeholder="เบอร์โทรศัพท์..." className="w-full p-4 bg-slate-50 rounded-2xl font-bold" value={customerInfo.phone} onChange={e => setCustomerInfo({...customerInfo, phone: e.target.value})} />
                <textarea placeholder="ที่อยู่จัดส่ง..." className="w-full p-4 bg-slate-50 rounded-[30px] font-bold min-h-[100px]" value={customerInfo.address} onChange={e => setCustomerInfo({...customerInfo, address: e.target.value})} />
                <button onClick={handleOrder} className="w-full bg-indigo-500 text-white py-5 rounded-3xl font-black uppercase text-sm tracking-widest shadow-2xl shadow-indigo-100">สั่งซื้อตอนนี้</button>
              </div>

              {/* REVIEWS */}
              <div className="pt-10 border-t border-slate-50">
                <h3 className="font-black text-lg mb-4 flex items-center gap-2 italic uppercase"><MessageSquare size={18}/> Reviews</h3>
                <div className="bg-slate-50 p-6 rounded-[35px] space-y-4 mb-6">
                  <div className="flex gap-1">
                    {[1,2,3,4,5].map(star => (
                      <button key={star} onClick={() => setReviewForm({...reviewForm, rating: star})}>
                        <Star size={20} className={reviewForm.rating >= star ? 'fill-yellow-400 text-yellow-400' : 'text-slate-200'} />
                      </button>
                    ))}
                  </div>
                  <textarea placeholder="บอกความประทับใจ..." className="w-full p-4 bg-white rounded-2xl text-xs font-bold min-h-[80px]" value={reviewForm.text} onChange={e => setReviewForm({...reviewForm, text: e.target.value})} />
                  <button onClick={async () => { playSound('success'); await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'reviews'), { productId: selectedProduct.id, userName: customerInfo.name || 'ลูกค้าทั่วไป', text: reviewForm.text, rating: reviewForm.rating, timestamp: Date.now() }); setReviewForm({text:'', rating:5}); }} className="w-full bg-slate-900 text-white py-3 rounded-2xl text-[10px] font-black uppercase tracking-widest">ส่งรีวิว</button>
                </div>
                <div className="space-y-4 pb-10">
                  {reviews.filter(r => r.productId === selectedProduct.id).map(r => (
                    <div key={r.id} className="bg-white p-5 rounded-3xl border border-slate-50 shadow-sm">
                      <div className="flex justify-between items-center mb-2"><span className="font-black text-[11px]">{r.userName}</span><div className="flex gap-0.5">{[...Array(r.rating)].map((_, i) => <Star key={i} size={8} className="fill-yellow-400 text-yellow-400" />)}</div></div>
                      <p className="text-xs text-slate-500">{r.text}</p>
                    </div>
                  ))}
                </div>
              </div>
            </div>
          </div>
        </div>
      )}

      {/* NAV BAR */}
      <nav className="fixed bottom-6 left-1/2 -translate-x-1/2 w-[90%] max-w-sm bg-white/70 backdrop-blur-2xl border border-white shadow-[0_15px_40px_-15px_rgba(0,0,0,0.1)] rounded-[35px] p-2.5 flex justify-between z-[100]">
        {!isAdminMode ? (
          <>
            <button onClick={() => handleMenuClick('shop')} className={`flex-1 py-4 rounded-[25px] flex flex-col items-center gap-1 ${view === 'shop' ? 'bg-slate-900 text-white shadow-xl scale-105' : 'text-slate-300'}`}>
              <Store size={22} /><span className="text-[9px] font-black uppercase tracking-widest">ร้านค้า</span>
            </button>
            <button onClick={() => handleMenuClick('my_orders')} className={`flex-1 py-4 rounded-[25px] flex flex-col items-center gap-1 ${view === 'my_orders' ? 'bg-slate-900 text-white shadow-xl scale-105' : 'text-slate-300'}`}>
              <ShoppingBag size={22} /><span className="text-[9px] font-black uppercase tracking-widest">สั่งซื้อ</span>
            </button>
          </>
        ) : (
          <>
            <button onClick={() => handleMenuClick('admin_orders')} className={`flex-1 py-4 rounded-[25px] flex flex-col items-center gap-1 ${view === 'admin_orders' ? 'bg-indigo-500 text-white shadow-xl scale-105' : 'text-slate-300'}`}>
              <LayoutGrid size={22} /><span className="text-[9px] font-black uppercase tracking-widest">ออเดอร์</span>
            </button>
            <button onClick={() => handleMenuClick('admin_inventory')} className={`flex-1 py-4 rounded-[25px] flex flex-col items-center gap-1 ${view === 'admin_inventory' ? 'bg-indigo-500 text-white shadow-xl scale-105' : 'text-slate-300'}`}>
              <Archive size={22} /><span className="text-[9px] font-black uppercase tracking-widest">คลัง</span>
            </button>
          </>
        )}
      </nav>

      <style dangerouslySetInnerHTML={{ __html: `
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .no-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
      `}} />
    </div>
  );
}
