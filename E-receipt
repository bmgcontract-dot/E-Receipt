import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInAnonymously, 
  onAuthStateChanged,
  signInWithCustomToken
} from 'firebase/auth';
import { 
  getFirestore, 
  collection, 
  addDoc, 
  onSnapshot, 
  query, 
  Timestamp,
  doc,
  setDoc
} from 'firebase/firestore';
import { 
  Printer, 
  Save, 
  History, 
  FileText, 
  Plus, 
  Trash2, 
  Search,
  Download,
  CheckCircle,
  Image as ImageIcon,
  X,
  Calendar,
  Settings,
  Edit
} from 'lucide-react';

// --- Firebase Configuration ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

// --- Constants & Data ---
const DEFAULT_SETTINGS = {
  nameTH: "นิติบุคคลหมู่บ้านจัดสรรไอลิฟ ทาวน์ ประชาอุทิศ 90",
  nameEN: "Juristic Person I-Leaf Town Pracha Uthit 90",
  address: "168/411 หมู่ 3 ซ.ประชาอุทิศ 90 ต.บ้านคลองสวน อ.พระสมุทรเจดีย์ จ.สมุทรปราการ 10290",
  receiptPrefix: "IL-",
  receivers: []
};

const EXPENSE_TYPES = [
  { id: 'common_fee', label: 'ค่าสาธารณูปโภค (Common Fee)' },
  { id: 'late_fee', label: 'ค่าปรับล่าช้า (Late Payment Fee)' },
  { id: 'water', label: 'ค่าน้ำประปา (Water Bill)' },
  { id: 'other', label: 'อื่นๆ (Other)' }
];

const PAYMENT_METHODS = [
  { id: 'cash', label: 'เงินสด (Cash)' },
  { id: 'transfer', label: 'โอนเงิน (Bank Transfer)' },
  { id: 'other', label: 'อื่นๆ (Other)' }
];

// --- Main Component ---
export default function App() {
  const [user, setUser] = useState(null);
  const [activeTab, setActiveTab] = useState('issue'); 
  const [receipts, setReceipts] = useState([]);
  const [loading, setLoading] = useState(true);
  
  // Toast State
  const [toast, setToast] = useState({ show: false, message: '', type: 'info' });

  // Settings State
  const [sysSettings, setSysSettings] = useState(DEFAULT_SETTINGS);
  const [settingsForm, setSettingsForm] = useState(DEFAULT_SETTINGS);
  
  // Global Defaults (Sync across all devices)
  const [globalDefaults, setGlobalDefaults] = useState({ paymentMethod: 'cash', receiver: '' });

  // Filter State
  const [filterDate, setFilterDate] = useState('');
  const [newReceiver, setNewReceiver] = useState('');

  // Form State
  const [formData, setFormData] = useState({
    id: null,
    receiptNo: '',
    createdAt: null,
    dateStr: '',
    timeStr: '',
    houseNo: '',
    name: '',
    items: [{ type: 'common_fee', description: '', amount: 0 }],
    paymentMethod: 'cash',
    transferDate: '',
    receiver: '',
    note: '',
    slipImage: null
  });
  const [nextReceiptId, setNextReceiptId] = useState('IL-001');
  const [showPreview, setShowPreview] = useState(false);
  const [previewData, setPreviewData] = useState(null);
  const [previewMode, setPreviewMode] = useState('new'); 
  const [pdfLibLoaded, setPdfLibLoaded] = useState(false);
  const [isGeneratingPdf, setIsGeneratingPdf] = useState(false);

  const showToast = (message, type = 'info') => {
    setToast({ show: true, message, type });
    setTimeout(() => setToast({ show: false, message: '', type: 'info' }), 4000);
  };

  // --- Auth & Initializing ---
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("Auth error:", err);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);

    const script = document.createElement('script');
    script.src = "https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js";
    script.onload = () => setPdfLibLoaded(true);
    document.body.appendChild(script);

    return () => {
        unsubscribe();
        if (document.body.contains(script)) {
            document.body.removeChild(script);
        }
    };
  }, []);

  // --- Real-time Data Sync ---
  useEffect(() => {
    if (!user) return;

    // 1. Fetch Settings
    const settingsRef = doc(db, 'artifacts', appId, 'public', 'data', 'settings', 'config');
    const unsubscribeSettings = onSnapshot(settingsRef, (docSnap) => {
      if (docSnap.exists()) {
        const data = docSnap.data();
        setSysSettings({ ...DEFAULT_SETTINGS, ...data });
        setSettingsForm({ ...DEFAULT_SETTINGS, ...data });
      }
    }, (err) => console.error("Settings sync error:", err));

    // 2. Fetch Global Defaults (receiver & payment method)
    const globalRef = doc(db, 'artifacts', appId, 'public', 'data', 'settings', 'globalDefaults');
    const unsubscribeGlobal = onSnapshot(globalRef, (docSnap) => {
      if (docSnap.exists()) {
        setGlobalDefaults(docSnap.data());
      }
    }, (err) => console.error("Global defaults sync error:", err));

    // 3. Fetch Receipts
    const q = query(collection(db, 'artifacts', appId, 'public', 'data', 'receipts'));
    const unsubscribeReceipts = onSnapshot(q, (snapshot) => {
      const data = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data(),
        createdAt: doc.data().createdAt?.toDate() || new Date(),
        paymentDate: doc.data().paymentDate ? doc.data().paymentDate.toDate() : null
      }));
      
      data.sort((a, b) => b.createdAt - a.createdAt);
      setReceipts(data);
      setLoading(false);
    }, (error) => {
      console.error("Error fetching receipts:", error);
      showToast("เกิดปัญหาในการดึงข้อมูล กรุณารีเฟรชหน้าเว็บ", "error");
      setLoading(false);
    });

    return () => {
      unsubscribeReceipts();
      unsubscribeSettings();
      unsubscribeGlobal();
    };
  }, [user]);

  // Recalculate next ID dynamically
  useEffect(() => {
    calculateNextId(receipts, sysSettings.receiptPrefix);
  }, [receipts, sysSettings.receiptPrefix]);

  // Robust Global Defaults Auto-fill (fixes the cross-device reset issue)
  useEffect(() => {
    setFormData(prev => {
        // อัปเดตเฉพาะกรณีที่เป็นฟอร์มว่างๆ (ยังไม่ได้พิมพ์อะไรลงไป)
        if (!prev.id && prev.houseNo === '' && prev.name === '') {
            return {
                ...prev,
                paymentMethod: globalDefaults.paymentMethod || 'cash',
                receiver: globalDefaults.receiver || ''
            };
        }
        return prev;
    });
  }, [globalDefaults]);

  // --- Logic ---
  const calculateNextId = (data, prefix = "IL-") => {
    if (data.length === 0) {
      setNextReceiptId(`${prefix}001`);
      return;
    }
    
    const matchingReceipts = data.filter(r => r.receiptNo && r.receiptNo.startsWith(prefix));
    if (matchingReceipts.length === 0) {
       setNextReceiptId(`${prefix}001`);
       return;
    }
    
    const numbers = matchingReceipts.map(r => {
      const strNum = r.receiptNo.substring(prefix.length);
      const num = parseInt(strNum, 10);
      return isNaN(num) ? 0 : num;
    });
    const max = Math.max(...numbers, 0);
    setNextReceiptId(`${prefix}${String(max + 1).padStart(3, '0')}`);
  };

  const getFilteredReceipts = () => {
    if (!filterDate) return receipts;
    return receipts.filter(r => {
      const rDate = new Date(r.createdAt);
      const fDate = new Date(filterDate);
      return (
        rDate.getFullYear() === fDate.getFullYear() &&
        rDate.getMonth() === fDate.getMonth() &&
        rDate.getDate() === fDate.getDate()
      );
    });
  };

  const filteredReceipts = getFilteredReceipts();

  const resetForm = () => {
    setFormData({
        id: null,
        receiptNo: '',
        createdAt: null,
        dateStr: '',
        timeStr: '',
        houseNo: '',
        name: '',
        items: [{ type: 'common_fee', description: '', amount: 0 }],
        paymentMethod: globalDefaults.paymentMethod || 'cash', 
        transferDate: '',
        receiver: globalDefaults.receiver || '', 
        note: '',
        slipImage: null
    });
  };

  const handleAddItem = () => {
    setFormData(prev => ({
      ...prev,
      items: [...prev.items, { type: 'common_fee', description: '', amount: 0 }]
    }));
  };

  const handleRemoveItem = (index) => {
    setFormData(prev => ({
      ...prev,
      items: prev.items.filter((_, i) => i !== index)
    }));
  };

  const handleItemChange = (index, field, value) => {
    const newItems = [...formData.items];
    newItems[index][field] = value;
    setFormData(prev => ({ ...prev, items: newItems }));
  };

  const handleFileChange = (e) => {
    const file = e.target.files[0];
    if (file) {
      if (file.size > 800 * 1024) {
          showToast("แจ้งเตือน: รูปภาพมีขนาดใหญ่ (เกิน 800KB) แนะนำให้ลดขนาดภาพก่อนแนบ", "warning");
      }
      const reader = new FileReader();
      reader.onloadend = () => {
        setFormData(prev => ({ ...prev, slipImage: reader.result }));
      };
      reader.readAsDataURL(file);
    }
  };

  const handleRemoveSlip = () => {
      setFormData(prev => ({ ...prev, slipImage: null }));
  };

  const calculateTotal = (items) => {
    return items.reduce((sum, item) => sum + (parseFloat(item.amount) || 0), 0);
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    const total = calculateTotal(formData.items);
    
    const receiptData = {
      ...formData,
      receiptNo: formData.id ? formData.receiptNo : nextReceiptId, 
      total: total,
      createdAt: formData.id ? formData.createdAt : new Date(), 
      status: 'completed',
      dateStr: formData.id ? formData.dateStr : new Date().toLocaleDateString('th-TH'),
      timeStr: formData.id ? formData.timeStr : new Date().toLocaleTimeString('th-TH')
    };

    setPreviewData(receiptData);
    setPreviewMode(formData.id ? 'edit' : 'new');
    setShowPreview(true);
  };

  const openHistoryReceipt = (receipt) => {
    setPreviewData(receipt);
    setPreviewMode('history');
    setShowPreview(true);
  };

  const editHistoryReceipt = (receipt) => {
    const itemsCopy = receipt.items ? JSON.parse(JSON.stringify(receipt.items)) : [{ type: 'common_fee', description: '', amount: 0 }];
    setFormData({
        id: receipt.id,
        receiptNo: receipt.receiptNo,
        createdAt: receipt.createdAt,
        dateStr: receipt.dateStr,
        timeStr: receipt.timeStr,
        houseNo: receipt.houseNo || '',
        name: receipt.name || '',
        items: itemsCopy,
        paymentMethod: receipt.paymentMethod || 'cash',
        transferDate: receipt.transferDate || '',
        receiver: receipt.receiver || '',
        note: receipt.note || '',
        slipImage: receipt.slipImage || null
    });
    setActiveTab('issue');
    setShowPreview(false);
  };

  const cancelEdit = () => {
    resetForm();
    showToast("ยกเลิกการแก้ไข ระบบกลับสู่โหมดออกใบเสร็จใหม่", "info");
  };

  const confirmSave = async () => {
    if (!user) return;

    try {
      const { id, ...cleanData } = previewData; // ถอด ID ออกก่อนบันทึกลงฟิลด์ของ Firestore
      const savePayload = {
        ...cleanData,
        createdAt: Timestamp.fromDate(previewData.createdAt),
        paymentDate: previewData.transferDate ? Timestamp.fromDate(new Date(previewData.transferDate)) : null
      };

      if (previewData.id) {
          // โหมดอัปเดต (แก้ไขใบเสร็จเดิม)
          const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'receipts', previewData.id);
          try {
            await setDoc(docRef, savePayload, { merge: true });
          } catch (innerError) {
            if (innerError.code === 'invalid-argument' || innerError.message.includes('size')) {
                 showToast("รูปภาพมีขนาดใหญ่เกินไป ระบบจะบันทึกข้อมูลโดยไม่มีรูปภาพ", "warning");
                 const payloadNoImage = { ...savePayload, slipImage: null };
                 await setDoc(docRef, payloadNoImage, { merge: true });
            } else {
                 throw innerError;
            }
          }
          showToast("อัปเดตข้อมูลใบเสร็จสำเร็จ!", "success");
      } else {
          // โหมดเพิ่มใบเสร็จใหม่
          try {
            await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'receipts'), savePayload);
          } catch (innerError) {
            if (innerError.code === 'invalid-argument' || innerError.message.includes('size')) {
                 showToast("รูปภาพมีขนาดใหญ่เกินไป ระบบจะบันทึกข้อมูลโดยไม่มีรูปภาพ", "warning");
                 const payloadNoImage = { ...savePayload, slipImage: null };
                 await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'receipts'), payloadNoImage);
            } else {
                 throw innerError;
            }
          }
          showToast("บันทึกใบเสร็จใหม่สำเร็จ!", "success");

          // แจ้งเตือน Cloud ให้จำค่า "ผู้รับเงิน" และ "วิธีชำระ" ล่าสุดไว้ใช้กับทุกอุปกรณ์
          await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'settings', 'globalDefaults'), {
            paymentMethod: savePayload.paymentMethod || 'cash',
            receiver: savePayload.receiver || ''
          }, { merge: true });
      }
      
      setShowPreview(false);
      setActiveTab('summary');
      resetForm(); 
      
    } catch (err) {
      console.error("Error saving receipt:", err);
      showToast("เกิดข้อผิดพลาดในการบันทึก: " + err.message, "error");
    }
  };

  const handleAddReceiver = () => {
    if (!newReceiver.trim()) return;
    setSettingsForm(prev => ({
      ...prev,
      receivers: [...(prev.receivers || []), newReceiver.trim()]
    }));
    setNewReceiver('');
  };

  const handleRemoveReceiver = (index) => {
    setSettingsForm(prev => ({
      ...prev,
      receivers: (prev.receivers || []).filter((_, i) => i !== index)
    }));
  };

  const handleSaveSettings = async (e) => {
    e.preventDefault();
    if (!user) {
        showToast("กรุณารอระบบยืนยันตัวตนสักครู่", "warning");
        return;
    }
    try {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'settings', 'config'), settingsForm);
      showToast("บันทึกการตั้งค่าระบบเรียบร้อยแล้ว", "success");
    } catch (err) {
      console.error("Error saving settings:", err);
      showToast("เกิดข้อผิดพลาดในการบันทึกการตั้งค่า: " + err.message, "error");
    }
  };

  const printReceipt = () => {
    window.print();
  };

  const downloadPDF = async (elementId, fileName, isHalfA4 = false) => {
    if (!pdfLibLoaded || typeof window.html2pdf === 'undefined') {
        showToast("กำลังโหลดระบบ PDF กรุณารอสักครู่...", "warning");
        return;
    }

    setIsGeneratingPdf(true);
    const element = document.getElementById(elementId);
    
    const opt = {
        margin: [5, 5, 5, 5], 
        filename: fileName,
        image: { type: 'jpeg', quality: 0.98 },
        html2canvas: { scale: 2, useCORS: true, logging: false },
        jsPDF: { 
            unit: 'mm', 
            format: isHalfA4 ? 'a5' : 'a4', 
            orientation: isHalfA4 ? 'landscape' : 'portrait' 
        },
        pagebreak: { mode: ['avoid-all', 'css', 'legacy'] }
    };

    try {
        await window.html2pdf().set(opt).from(element).save();
        showToast("ดาวน์โหลด PDF สำเร็จ", "success");
    } catch (error) {
        console.error("PDF Gen Error:", error);
        showToast("ขออภัย เกิดข้อผิดพลาดในการสร้างไฟล์ PDF", "error");
    } finally {
        setIsGeneratingPdf(false);
    }
  };

  if (loading) return <div className="flex items-center justify-center h-screen text-gray-500">Loading System...</div>;

  return (
    <div className="min-h-screen bg-gray-50 text-gray-800 font-sans pb-10">
      
      {toast.show && (
        <div className={`fixed top-4 right-4 z-[9999] px-6 py-3 rounded-lg shadow-xl font-medium text-white flex items-center transition-all animate-bounce ${
          toast.type === 'error' ? 'bg-red-600' : 
          toast.type === 'warning' ? 'bg-orange-500' : 'bg-green-600'
        }`}>
          {toast.type === 'success' && <CheckCircle className="w-5 h-5 mr-2" />}
          {toast.message}
        </div>
      )}

      <nav className="bg-red-900 text-white shadow-lg print:hidden">
        <div className="max-w-6xl mx-auto px-4 py-4 flex justify-between items-center">
          <div className="flex items-center space-x-2">
            <FileText className="h-6 w-6" />
            <div>
              <h1 className="text-lg font-bold">ระบบบันทึกใบเสร็จรับเงิน</h1>
              <p className="text-xs text-orange-200">{sysSettings.nameEN}</p>
            </div>
          </div>
          <div className="flex space-x-4">
            <button 
              onClick={() => setActiveTab('issue')}
              className={`px-4 py-2 rounded-lg transition ${activeTab === 'issue' ? 'bg-white text-red-900 font-bold' : 'hover:bg-red-800'}`}
            >
              ออกใบเสร็จ (Issue)
            </button>
            <button 
              onClick={() => setActiveTab('summary')}
              className={`px-4 py-2 rounded-lg transition ${activeTab === 'summary' ? 'bg-white text-red-900 font-bold' : 'hover:bg-red-800'}`}
            >
              สรุปยอด (Summary)
            </button>
            <button 
              onClick={() => setActiveTab('settings')}
              className={`px-4 py-2 rounded-lg transition ${activeTab === 'settings' ? 'bg-white text-red-900 font-bold' : 'hover:bg-red-800'}`}
            >
              <Settings className="w-5 h-5 inline-block mr-1" /> ตั้งค่า (Settings)
            </button>
          </div>
        </div>
      </nav>

      <main className="max-w-6xl mx-auto px-4 py-8 print:hidden">
        
        {activeTab === 'issue' && (
          <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
            <div className="lg:col-span-2 bg-white p-6 rounded-xl shadow-sm border border-gray-200">
              <h2 className="text-xl font-bold mb-6 flex items-center text-red-900 border-b pb-2">
                {formData.id ? <Edit className="w-5 h-5 mr-2" /> : <Plus className="w-5 h-5 mr-2" />}
                {formData.id ? `แก้ไขใบเสร็จ / Edit Receipt: ${formData.receiptNo}` : 'ออกใบเสร็จรับเงินใหม่ / New Receipt'}
                {!formData.id && (
                  <span className="ml-auto text-sm font-normal text-gray-500 bg-gray-100 px-2 py-1 rounded">
                    Next: {nextReceiptId}
                  </span>
                )}
              </h2>

              <form onSubmit={handleSubmit} className="space-y-6">
                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">บ้านเลขที่ / House No.</label>
                    <input 
                      required
                      type="text" 
                      className="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-orange-500 outline-none transition"
                      placeholder="e.g. 168/99"
                      value={formData.houseNo}
                      onChange={e => setFormData({...formData, houseNo: e.target.value})}
                    />
                  </div>
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">ชื่อ-นามสกุล / Name</label>
                    <input 
                      required
                      type="text" 
                      className="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-orange-500 outline-none transition"
                      placeholder="เจ้าของบ้าน / Owner Name"
                      value={formData.name}
                      onChange={e => setFormData({...formData, name: e.target.value})}
                    />
                  </div>
                </div>

                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-2">รายละเอียด / Details</label>
                  <div className="bg-gray-50 p-4 rounded-lg border border-gray-200 space-y-3">
                    {formData.items.map((item, idx) => (
                      <div key={idx} className="flex flex-col md:flex-row gap-2 items-start md:items-center">
                        <div className="flex-1 w-full">
                          <select 
                            className="w-full p-2 border border-gray-300 rounded-md text-sm"
                            value={item.type}
                            onChange={(e) => handleItemChange(idx, 'type', e.target.value)}
                          >
                            {EXPENSE_TYPES.map(type => (
                              <option key={type.id} value={type.id}>{type.label}</option>
                            ))}
                          </select>
                        </div>
                        <div className="flex-1 w-full">
                          <input 
                            type="text" 
                            className="w-full p-2 border border-gray-300 rounded-md text-sm"
                            placeholder="ระบุรายละเอียดเพิ่มเติม (ถ้ามี)"
                            value={item.description}
                            onChange={(e) => handleItemChange(idx, 'description', e.target.value)}
                          />
                        </div>
                        <div className="w-full md:w-32">
                          <input 
                            type="number" 
                            min="0"
                            step="0.01"
                            className="w-full p-2 border border-gray-300 rounded-md text-sm text-right"
                            placeholder="0.00"
                            value={item.amount}
                            onChange={(e) => handleItemChange(idx, 'amount', parseFloat(e.target.value) || 0)}
                          />
                        </div>
                        {formData.items.length > 1 && (
                          <button 
                            type="button" 
                            onClick={() => handleRemoveItem(idx)}
                            className="text-red-500 hover:bg-red-50 p-2 rounded"
                          >
                            <Trash2 className="w-4 h-4" />
                          </button>
                        )}
                      </div>
                    ))}
                    <button 
                      type="button" 
                      onClick={handleAddItem}
                      className="text-sm text-orange-600 font-medium hover:underline flex items-center"
                    >
                      <Plus className="w-4 h-4 mr-1" /> เพิ่มรายการ / Add Item
                    </button>
                  </div>
                  <div className="flex justify-end mt-2">
                    <div className="text-lg font-bold text-red-900">
                      รวมเป็นเงิน / Total: {calculateTotal(formData.items).toLocaleString(undefined, {minimumFractionDigits: 2})} บาท
                    </div>
                  </div>
                </div>

                <div className="grid grid-cols-1 md:grid-cols-2 gap-4 border-t pt-4">
                  <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">ชำระโดย / Payment Method</label>
                    <div className="flex gap-4 mt-2">
                      {PAYMENT_METHODS.map(method => (
                        <label key={method.id} className="flex items-center cursor-pointer">
                          <input 
                            type="radio" 
                            name="paymentMethod" 
                            value={method.id}
                            checked={formData.paymentMethod === method.id}
                            onChange={(e) => setFormData({...formData, paymentMethod: e.target.value})}
                            className="mr-2"
                          />
                          <span className="text-sm">{method.label.split(' ')[0]}</span>
                        </label>
                      ))}
                    </div>
                  </div>

                  {formData.paymentMethod === 'transfer' && (
                     <div>
                       <label className="block text-sm font-medium text-gray-700 mb-1">วันที่โอน / Transfer Date</label>
                       <input 
                        type="date" 
                        required
                        className="w-full p-2 border border-gray-300 rounded-md"
                        value={formData.transferDate}
                        onChange={(e) => setFormData({...formData, transferDate: e.target.value})}
                       />
                     </div>
                  )}
                </div>

                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                   <div>
                      <label className="block text-sm font-medium text-gray-700 mb-1">แนบสลิป (รูปภาพ) / Attach Slip (Image)</label>
                      <div className="w-full">
                        {!formData.slipImage ? (
                          <label className="flex flex-col items-center justify-center w-full h-24 border-2 border-gray-300 border-dashed rounded-lg cursor-pointer bg-gray-50 hover:bg-gray-100">
                              <div className="flex flex-col items-center justify-center pt-5 pb-6">
                                  <ImageIcon className="w-6 h-6 text-gray-400 mb-1" />
                                  <p className="text-xs text-gray-500">Click to upload image</p>
                              </div>
                              <input 
                                  type="file" 
                                  className="hidden" 
                                  accept="image/*"
                                  onChange={handleFileChange} 
                              />
                          </label>
                        ) : (
                          <div className="relative border rounded-lg p-2 bg-gray-50 flex justify-center">
                              <img src={formData.slipImage} alt="Slip Preview" className="h-40 object-contain" />
                              <button 
                                type="button"
                                onClick={handleRemoveSlip}
                                className="absolute top-2 right-2 bg-red-500 text-white p-1 rounded-full shadow hover:bg-red-600"
                              >
                                <X className="w-4 h-4" />
                              </button>
                          </div>
                        )}
                      </div>
                   </div>
                   <div>
                    <label className="block text-sm font-medium text-gray-700 mb-1">ผู้รับเงิน / Receiver Name (Optional)</label>
                    {sysSettings.receivers && sysSettings.receivers.length > 0 ? (
                      <select 
                        className="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-orange-500 outline-none"
                        value={formData.receiver}
                        onChange={e => setFormData({...formData, receiver: e.target.value})}
                      >
                        <option value="">-- ไม่ระบุ / None --</option>
                        {sysSettings.receivers.map((r, i) => (
                          <option key={i} value={r}>{r}</option>
                        ))}
                      </select>
                    ) : (
                      <input 
                        type="text" 
                        className="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-orange-500 outline-none"
                        placeholder="ชื่อเจ้าหน้าที่ (ถ้ามี)"
                        value={formData.receiver}
                        onChange={e => setFormData({...formData, receiver: e.target.value})}
                      />
                    )}
                  </div>
                </div>

                <div className="pt-4 flex gap-4">
                  <button 
                    type="submit"
                    className="flex-1 bg-orange-600 text-white py-3 rounded-lg font-bold shadow-md hover:bg-orange-700 transition flex justify-center items-center"
                  >
                    <CheckCircle className="w-5 h-5 mr-2" />
                    {formData.id ? 'ตรวจสอบการแก้ไข / Preview Edit' : 'สร้างและตรวจสอบ / Create & Preview'}
                  </button>
                  
                  {formData.id && (
                     <button 
                       type="button"
                       onClick={cancelEdit}
                       className="px-6 bg-gray-300 text-gray-700 py-3 rounded-lg font-bold shadow-sm hover:bg-gray-400 transition"
                     >
                       ยกเลิก / Cancel
                     </button>
                  )}
                </div>
              </form>
            </div>

            <div className="lg:col-span-1">
              <div className="bg-red-50 p-6 rounded-xl border border-red-100 text-red-900 mb-6">
                <h3 className="font-bold mb-2 flex items-center"><History className="w-4 h-4 mr-2"/> Recent Receipts</h3>
                <div className="space-y-3">
                  {receipts.slice(0, 5).map(r => (
                    <div key={r.id} className="bg-white p-3 rounded shadow-sm text-sm flex justify-between cursor-pointer hover:bg-gray-50 transition" onClick={() => openHistoryReceipt(r)}>
                      <div>
                        <div className="font-bold flex items-center">{r.receiptNo} <Search className="w-3 h-3 ml-1 text-orange-400"/></div>
                        <div className="text-xs text-gray-500">{r.houseNo}</div>
                      </div>
                      <div className="text-right">
                        <div className="font-bold text-orange-600">{r.total.toLocaleString()}.-</div>
                        <div className="text-xs text-gray-400">{new Date(r.createdAt).toLocaleDateString()}</div>
                      </div>
                    </div>
                  ))}
                  {receipts.length === 0 && <p className="text-sm opacity-60">ยังไม่มีข้อมูล / No records found</p>}
                </div>
              </div>
            </div>
          </div>
        )}

        {/* --- SUMMARY TAB --- */}
        {activeTab === 'summary' && (
          <div className="bg-white p-6 rounded-xl shadow-sm border border-gray-200">
            <div className="flex flex-col md:flex-row justify-between items-start md:items-center mb-6 gap-4">
                <h2 className="text-xl font-bold flex items-center text-red-900">
                <History className="w-5 h-5 mr-2" />
                สรุปการรับเงิน / Receipts Summary
                </h2>
                
                <button
                    onClick={() => downloadPDF('summary-report-content', `Summary-Report-${filterDate || 'All'}.pdf`, false)}
                    disabled={!pdfLibLoaded || isGeneratingPdf}
                    className={`px-4 py-2 rounded text-white flex items-center shadow-sm text-sm ${!pdfLibLoaded || isGeneratingPdf ? 'bg-gray-400 cursor-not-allowed' : 'bg-red-600 hover:bg-red-700'}`}
                >
                    <Download className="w-4 h-4 mr-2" />
                    Download PDF Report
                </button>
            </div>
            
            <div className="flex flex-wrap gap-4 mb-6 items-end bg-gray-50 p-4 rounded-lg border border-gray-200">
              <div>
                <label className="block text-xs font-bold text-gray-700 mb-1 flex items-center"><Calendar className="w-3 h-3 mr-1"/> วันที่ทำรายการ / Date Filter</label>
                <input 
                    type="date" 
                    className="p-2 border rounded-md text-sm shadow-sm" 
                    value={filterDate}
                    onChange={(e) => setFilterDate(e.target.value)}
                />
                {filterDate && (
                    <button 
                        onClick={() => setFilterDate('')} 
                        className="ml-2 text-xs text-red-500 hover:underline"
                    >
                        ล้างค่า / Clear
                    </button>
                )}
              </div>
              <div className="flex-grow"></div>
              <div className="flex gap-6 text-right">
                <div>
                    <div className="text-xs text-gray-500">จำนวนใบเสร็จ / Count</div>
                    <div className="text-xl font-bold text-gray-800">
                        {filteredReceipts.length} รายการ
                    </div>
                </div>
                <div>
                    <div className="text-xs text-gray-500">รวมยอดสุทธิ / Total Amount</div>
                    <div className="text-xl font-bold text-green-600">
                        {filteredReceipts.reduce((sum, r) => sum + r.total, 0).toLocaleString(undefined, {minimumFractionDigits: 2})} THB
                    </div>
                </div>
              </div>
            </div>

            <div className="flex justify-center mt-8 overflow-x-auto">
              <div id="summary-report-content" className="bg-white p-8 shadow-md border border-gray-300 w-full max-w-[210mm] mx-auto">
                  <div className="mb-6 text-center">
                       <h3 className="text-xl font-bold mb-2 text-gray-900">{sysSettings.nameTH}</h3>
                       <p className="text-gray-700 font-medium">รายงานสรุปการรับเงิน {filterDate ? `ประจำวันที่ ${new Date(filterDate).toLocaleDateString('th-TH')}` : '(ทั้งหมด)'}</p>
                  </div>
                  
                  <table className="w-full text-sm text-left text-gray-800 border-collapse border border-gray-400">
                      <thead className="bg-gray-100 text-gray-800 uppercase font-bold text-center">
                      <tr>
                          <th className="px-4 py-3 border border-gray-400">No.</th>
                          <th className="px-4 py-3 border border-gray-400">Date</th>
                          <th className="px-4 py-3 border border-gray-400">House No</th>
                          <th className="px-4 py-3 border border-gray-400">Name</th>
                          <th className="px-4 py-3 border border-gray-400">Type</th>
                          <th className="px-4 py-3 border border-gray-400">Amount</th>
                      </tr>
                      </thead>
                      <tbody>
                      {filteredReceipts.map((r, index) => (
                          <tr key={r.id} className="hover:bg-gray-50">
                          <td className="px-4 py-2 border border-gray-400 text-center">{index + 1}</td>
                          <td className="px-4 py-2 border border-gray-400 text-center">{new Date(r.createdAt).toLocaleDateString('th-TH')}</td>
                          <td className="px-4 py-2 border border-gray-400 text-center">
                              <button 
                                onClick={() => openHistoryReceipt(r)}
                                className="hover:underline hover:text-orange-800 text-orange-600 font-bold transition flex items-center justify-center w-full"
                                title="ดูรายละเอียดใบเสร็จ"
                              >
                                {r.houseNo}
                              </button>
                          </td>
                          <td className="px-4 py-2 border border-gray-400 text-left">{r.name}</td>
                          <td className="px-4 py-2 border border-gray-400 text-center">
                              {EXPENSE_TYPES.find(t => t.id === r.items[0]?.type)?.label.split(' ')[0] || 'N/A'}
                          </td>
                          <td className="px-4 py-2 border border-gray-400 text-right font-medium">{r.total.toLocaleString(undefined, {minimumFractionDigits: 2})}</td>
                          </tr>
                      ))}
                      {filteredReceipts.length === 0 && (
                          <tr><td colSpan="6" className="text-center py-8 text-gray-400 border border-gray-400">ไม่พบข้อมูล / No data found</td></tr>
                      )}
                      </tbody>
                      {filteredReceipts.length > 0 && (
                          <tfoot>
                              <tr className="bg-gray-100 font-bold">
                                  <td colSpan="5" className="px-4 py-3 text-right border border-gray-400">รวมทั้งสิ้น / Grand Total</td>
                                  <td className="px-4 py-3 text-right text-red-700 border border-gray-400">
                                      {filteredReceipts.reduce((sum, r) => sum + r.total, 0).toLocaleString(undefined, {minimumFractionDigits: 2})}
                                  </td>
                              </tr>
                          </tfoot>
                      )}
                  </table>
                  
                  <div className="mt-8 text-center text-xs text-gray-400">
                     * เอกสารรายงานนี้สร้างขึ้นโดยระบบอัตโนมัติ / Generated by System
                  </div>
              </div>
            </div>
          </div>
        )}

        {/* --- SETTINGS TAB --- */}
        {activeTab === 'settings' && (
          <div className="bg-white p-6 rounded-xl shadow-sm border border-gray-200 max-w-2xl mx-auto">
            <h2 className="text-xl font-bold mb-6 flex items-center text-red-900 border-b pb-2">
              <Settings className="w-5 h-5 mr-2" />
              ตั้งค่าระบบ / System Settings
            </h2>
            
            <form onSubmit={handleSaveSettings} className="space-y-6">
              <div>
                <label className="block text-sm font-medium text-gray-700 mb-1">ชื่อหน่วยงาน/บริษัท (ภาษาไทย)</label>
                <input 
                  required
                  type="text" 
                  className="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-orange-500 outline-none"
                  value={settingsForm.nameTH}
                  onChange={e => setSettingsForm({...settingsForm, nameTH: e.target.value})}
                />
              </div>
              
              <div>
                <label className="block text-sm font-medium text-gray-700 mb-1">ชื่อหน่วยงาน/บริษัท (ภาษาอังกฤษ)</label>
                <input 
                  required
                  type="text" 
                  className="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-orange-500 outline-none"
                  value={settingsForm.nameEN}
                  onChange={e => setSettingsForm({...settingsForm, nameEN: e.target.value})}
                />
              </div>

              <div>
                <label className="block text-sm font-medium text-gray-700 mb-1">ที่อยู่ (Address)</label>
                <textarea 
                  required
                  rows="3"
                  className="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-orange-500 outline-none"
                  value={settingsForm.address}
                  onChange={e => setSettingsForm({...settingsForm, address: e.target.value})}
                ></textarea>
              </div>

              <div>
                <label className="block text-sm font-medium text-gray-700 mb-1">อักษรนำหน้าเลขที่ใบเสร็จ (Receipt Prefix)</label>
                <input 
                  required
                  type="text" 
                  className="w-full p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-orange-500 outline-none"
                  placeholder="e.g. IL-"
                  value={settingsForm.receiptPrefix}
                  onChange={e => setSettingsForm({...settingsForm, receiptPrefix: e.target.value})}
                />
                <p className="text-xs text-gray-500 mt-1">เช่น IL-, REC-, INV- (ระบบจะรันเลขต่อจากอักษรนี้อัตโนมัติ เช่น {settingsForm.receiptPrefix || 'IL-'}001)</p>
              </div>

              <div className="border-t border-gray-200 pt-4">
                <label className="block text-sm font-medium text-gray-700 mb-2">รายชื่อเจ้าหน้าที่ผู้รับเงิน (Receivers List)</label>
                <div className="flex gap-2 mb-3">
                  <input 
                    type="text" 
                    className="flex-1 p-2 border border-gray-300 rounded-md focus:ring-2 focus:ring-orange-500 outline-none"
                    placeholder="เพิ่มชื่อเจ้าหน้าที่..."
                    value={newReceiver}
                    onChange={e => setNewReceiver(e.target.value)}
                    onKeyDown={e => { if (e.key === 'Enter') { e.preventDefault(); handleAddReceiver(); } }}
                  />
                  <button 
                    type="button"
                    onClick={handleAddReceiver}
                    className="bg-orange-100 text-orange-700 px-4 py-2 rounded-md font-medium hover:bg-orange-200 transition whitespace-nowrap"
                  >
                    เพิ่ม (Add)
                  </button>
                </div>
                <div className="space-y-2 max-h-40 overflow-y-auto pr-2">
                  {(settingsForm.receivers || []).map((r, i) => (
                    <div key={i} className="flex justify-between items-center bg-gray-50 p-2 rounded border border-gray-200">
                      <span className="text-sm font-medium">{r}</span>
                      <button 
                        type="button" 
                        onClick={() => handleRemoveReceiver(i)}
                        className="text-red-500 hover:bg-red-100 p-1.5 rounded transition"
                        title="ลบชื่อนี้"
                      >
                        <Trash2 className="w-4 h-4" />
                      </button>
                    </div>
                  ))}
                  {(!settingsForm.receivers || settingsForm.receivers.length === 0) && (
                    <p className="text-sm text-gray-400 text-center py-2">ยังไม่มีรายชื่อ (ผู้ใช้จะพิมพ์ชื่อเองในหน้าออกใบเสร็จ)</p>
                  )}
                </div>
              </div>

              <div className="pt-4 border-t border-gray-200">
                <button 
                  type="submit"
                  className="w-full bg-orange-600 text-white py-3 rounded-lg font-bold shadow-md hover:bg-orange-700 transition flex justify-center items-center"
                >
                  <Save className="w-5 h-5 mr-2" />
                  บันทึกการตั้งค่า / Save Settings
                </button>
              </div>
            </form>
          </div>
        )}
      </main>

      {/* --- PREVIEW & PRINT MODAL --- */}
      {showPreview && previewData && (
        <div className="fixed inset-0 bg-black bg-opacity-50 z-50 flex items-center justify-center p-4 print:p-0 print:bg-white print:static print:block">
          
          <div className="bg-white rounded-lg shadow-2xl w-full max-w-3xl overflow-hidden flex flex-col max-h-[90vh] print:shadow-none print:max-w-none print:max-h-none print:w-full print:overflow-visible">
            
            <div className="bg-gray-100 p-4 border-b flex justify-between items-center print:hidden">
              <h3 className="font-bold text-lg">
                {previewMode === 'history' ? 'รายละเอียดใบเสร็จ / Receipt Details' : 
                 previewMode === 'edit' ? 'ตรวจสอบการแก้ไข / Preview Edit' : 'ตรวจสอบความถูกต้อง / Preview'}
              </h3>
              <div className="flex space-x-2">
                {previewMode === 'history' && (
                    <>
                      <button 
                          onClick={() => editHistoryReceipt(previewData)}
                          className="px-4 py-2 rounded text-blue-600 bg-blue-50 hover:bg-blue-100 flex items-center transition"
                      >
                          <Edit className="w-4 h-4 mr-1"/> แก้ไขข้อมูล / Edit
                      </button>
                      <button 
                          onClick={() => setShowPreview(false)}
                          className="px-4 py-2 rounded text-gray-600 hover:bg-gray-200 flex items-center"
                      >
                          <X className="w-4 h-4 mr-1"/> ปิด / Close
                      </button>
                    </>
                )}

                {(previewMode === 'new' || previewMode === 'edit') && (
                    <button 
                        onClick={() => setShowPreview(false)}
                        className="px-4 py-2 rounded text-gray-600 hover:bg-gray-200"
                    >
                        กลับไปหน้าฟอร์ม / Back
                    </button>
                )}

                <button 
                  onClick={() => downloadPDF('receipt-area', `Receipt-${previewData.receiptNo}.pdf`, true)}
                  disabled={!pdfLibLoaded || isGeneratingPdf}
                  className={`px-4 py-2 rounded text-white flex items-center shadow-sm ${!pdfLibLoaded || isGeneratingPdf ? 'bg-gray-400 cursor-not-allowed' : 'bg-red-600 hover:bg-red-700'}`}
                >
                  {isGeneratingPdf ? (
                    <span className="flex items-center">Generating...</span>
                  ) : (
                    <span className="flex items-center"><Download className="w-4 h-4 mr-2" /> Download PDF</span>
                  )}
                </button>

                <button 
                  onClick={printReceipt}
                  className="px-4 py-2 rounded bg-gray-800 text-white hover:bg-gray-700 flex items-center"
                >
                  <Printer className="w-4 h-4 mr-2" /> Print
                </button>
                
                {(previewMode === 'new' || previewMode === 'edit') && (
                    <button 
                    onClick={confirmSave}
                    className="px-4 py-2 rounded bg-green-600 text-white hover:bg-green-700 flex items-center shadow-lg transform hover:-translate-y-0.5 transition"
                    >
                    <Save className="w-4 h-4 mr-2" /> บันทึก / Save
                    </button>
                )}
              </div>
            </div>

            <div className="overflow-y-auto p-4 md:p-8 print:p-0 print:overflow-visible bg-gray-50">
              <div id="receipt-area" className="border border-gray-300 p-6 min-h-[130mm] relative bg-white text-black print:border-none print:min-h-0 shadow-lg print:shadow-none mx-auto w-full max-w-[210mm]">
                
                <div className="text-center mb-4">
                  <h1 className="text-xl font-bold mb-1">{sysSettings.nameTH}</h1>
                  <h2 className="text-sm font-semibold text-gray-600 mb-1">{sysSettings.nameEN}</h2>
                  <p className="text-xs text-gray-500 max-w-md mx-auto">{sysSettings.address}</p>
                  <div className="mt-2 border-2 border-double border-gray-800 inline-block px-6 py-1">
                    <span className="font-bold text-base">ใบเสร็จรับเงิน (ชั่วคราว) / TEMPORARY RECEIPT</span>
                  </div>
                </div>

                <div className="flex justify-between text-sm mb-4">
                   <div className="w-2/3">
                      <div className="grid grid-cols-3 gap-y-1">
                         <div className="font-bold">ได้รับเงินจาก (Received From):</div>
                         <div className="col-span-2 border-b border-dotted border-gray-400">{previewData.name}</div>
                         
                         <div className="font-bold">บ้านเลขที่ (House No.):</div>
                         <div className="col-span-2 border-b border-dotted border-gray-400">{previewData.houseNo}</div>
                      </div>
                   </div>
                   <div className="w-1/3 text-right">
                      <div className="grid grid-cols-2 gap-y-1 justify-end">
                         <div className="font-bold">เลขที่ (No.):</div>
                         <div className="text-red-600 font-mono font-bold">{previewData.receiptNo}</div>
                         
                         <div className="font-bold">วันที่ (Date):</div>
                         <div>{previewData.dateStr}</div>
                         
                         <div className="font-bold">เวลา (Time):</div>
                         <div>{previewData.timeStr}</div>
                      </div>
                   </div>
                </div>

                <table className="w-full text-sm border-collapse border border-gray-800 mb-4">
                  <thead>
                    <tr className="bg-gray-100">
                       <th className="border border-gray-800 p-1 w-16 text-center">ลำดับ<br/>No.</th>
                       <th className="border border-gray-800 p-1 text-center">รายการ<br/>Description</th>
                       <th className="border border-gray-800 p-1 w-32 text-center">จำนวนเงิน<br/>Amount (THB)</th>
                    </tr>
                  </thead>
                  <tbody>
                    {previewData.items.map((item, i) => (
                      <tr key={i}>
                        <td className="border-l border-r border-gray-800 p-1 text-center align-top h-6">{i + 1}</td>
                        <td className="border-l border-r border-gray-800 p-1 align-top">
                          <span className="font-semibold">{EXPENSE_TYPES.find(t => t.id === item.type)?.label}</span>
                          {item.description && <div className="text-xs text-gray-600 ml-2">- {item.description}</div>}
                        </td>
                        <td className="border-l border-r border-gray-800 p-1 text-right align-top">{parseFloat(item.amount).toLocaleString(undefined, {minimumFractionDigits: 2})}</td>
                      </tr>
                    ))}
                    {[...Array(Math.max(0, 4 - previewData.items.length))].map((_, i) => (
                       <tr key={`empty-${i}`}>
                         <td className="border-l border-r border-gray-800 p-1 h-6"></td>
                         <td className="border-l border-r border-gray-800 p-1"></td>
                         <td className="border-l border-r border-gray-800 p-1"></td>
                       </tr>
                    ))}
                  </tbody>
                  <tfoot>
                    <tr className="bg-gray-100 font-bold">
                       <td colSpan="2" className="border border-gray-800 p-1 text-right">รวมเงินทั้งสิ้น / Grand Total</td>
                       <td className="border border-gray-800 p-1 text-right">{previewData.total.toLocaleString(undefined, {minimumFractionDigits: 2})}</td>
                    </tr>
                  </tfoot>
                </table>

                <div className="text-sm mb-4">
                   <div className="font-bold mb-1 underline">การชำระเงิน (Payment Method)</div>
                   <div className="flex gap-6">
                      <div className="flex items-center">
                         <div className={`w-4 h-4 border border-black mr-2 flex items-center justify-center`}>
                            {previewData.paymentMethod === 'cash' && <div className="w-2 h-2 bg-black rounded-full"></div>}
                         </div>
                         เงินสด (Cash)
                      </div>
                      <div className="flex items-center">
                         <div className={`w-4 h-4 border border-black mr-2 flex items-center justify-center`}>
                            {previewData.paymentMethod === 'transfer' && <div className="w-2 h-2 bg-black rounded-full"></div>}
                         </div>
                         โอนเงิน (Transfer)
                         {previewData.paymentMethod === 'transfer' && previewData.transferDate && (
                            <span className="ml-2 border-b border-dotted border-black px-1">
                               วันที่: {new Date(previewData.transferDate).toLocaleDateString('th-TH')}
                            </span>
                         )}
                      </div>
                      <div className="flex items-center">
                         <div className={`w-4 h-4 border border-black mr-2 flex items-center justify-center`}>
                            {previewData.paymentMethod === 'other' && <div className="w-2 h-2 bg-black rounded-full"></div>}
                         </div>
                         อื่นๆ (Other)
                      </div>
                   </div>
                </div>

                <div className="flex justify-between text-center mt-6 px-8 page-break-inside-avoid">
                   <div className="w-1/3">
                      <div className="border-b border-black mb-1 h-6"></div>
                      <div className="text-sm">ผู้ชำระเงิน / Payer</div>
                   </div>
                   <div className="w-1/3">
                      <div className="border-b border-black mb-1 h-6 flex items-end justify-center font-script text-base">
                        {previewData.receiver ? previewData.receiver : ''}
                      </div>
                      <div className="text-sm">ผู้รับเงิน / Receiver</div>
                   </div>
                </div>

                {previewData.slipImage && (
                  <div className="mt-4 border-t border-gray-300 pt-2 page-break-inside-avoid">
                    <div className="font-bold mb-1 text-xs text-gray-500">หลักฐานการชำระเงิน / Payment Slip (Attached):</div>
                    <div className="flex justify-center">
                       <img 
                          src={previewData.slipImage} 
                          alt="Attached Slip" 
                          className="max-w-[80%] max-h-[150px] object-contain border border-gray-200"
                        />
                    </div>
                  </div>
                )}

                <div className="mt-4 text-center text-xs text-gray-400">
                   * เอกสารนี้สร้างขึ้นโดยระบบอัตโนมัติ / Generated by System
                </div>

              </div>
            </div>
          </div>
        </div>
      )}

      {/* Print Styles */}
      <style>{`
        @media print {
          body * {
            visibility: hidden;
          }
          #receipt-area, #receipt-area * {
            visibility: visible;
          }
          #summary-report-content, #summary-report-content * {
             visibility: visible;
          }
          #receipt-area, #summary-report-content {
            position: absolute;
            left: 0;
            top: 0;
            width: 100%;
            margin: 0;
            padding: 0;
            border: none;
            box-shadow: none;
            background: white;
          }
          .fixed, .overflow-y-auto, .max-h-\[90vh\] {
            position: static !important;
            overflow: visible !important;
            height: auto !important;
            max-height: none !important;
            width: auto !important;
          }
          nav, button, .lucide, .print\:hidden {
            display: none !important;
          }
          .page-break-inside-avoid { 
            page-break-inside: avoid; 
          }
        }
      `}</style>
    </div>
  );
}
