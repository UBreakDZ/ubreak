import { useState, useEffect, createContext, useContext } from "react";
import { createClient } from "@supabase/supabase-js";

const SUPABASE_URL = "https://ipgljepqvuuvnhovynkc.supabase.co";
const SUPABASE_ANON = "sb_publishable_J3KQN8i-2Se45CBocugMVw_KLEu5pHH";
const supabase = createClient(SUPABASE_URL, SUPABASE_ANON);

const C = {
  green:"#4E9A0E", greenDark:"#2E6008", greenLight:"#7DC142",
  greenGhost:"#EDF7E0", cream:"#F5FAF0", white:"#FFFFFF",
  text:"#182E06", textMuted:"#6B8F5A", border:"#D0E9B8",
  danger:"#D94F4F", dangerLight:"#FDEAEA",
  warning:"#E08A1A", successLight:"#E3F7EE",
  success:"#2E9B5A", sidebar:"#1A3A05", sidebarText:"rgba(255,255,255,0.7)",
};

const CATEGORIES = ["Sandwichs","Pizzas","Box Salade","Box Riz","Box Pâtes","Boissons","Desserts"];
const StoreCtx = createContext(null);
const useStore = () => useContext(StoreCtx);

function StoreProvider({ children }) {
  const [session, setSession] = useState(null);
  const [profile, setProfile] = useState(null);
  const [isAdmin, setIsAdmin] = useState(false);
  const [menu, setMenu] = useState([]);
  const [ingredients, setIngredients] = useState({ breads:[], proteins:[], veggies:[], sauces:[] });
  const [stock, setStock] = useState([]);
  const [orders, setOrders] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    supabase.auth.getSession().then(({ data: { session } }) => {
      setSession(session);
      if (session) loadUserData(session.user.id);
      else setLoading(false);
    });
    const { data: { subscription } } = supabase.auth.onAuthStateChange((_e, session) => {
      setSession(session);
      if (session) loadUserData(session.user.id);
      else { setProfile(null); setIsAdmin(false); setLoading(false); }
    });
    return () => subscription.unsubscribe();
  }, []);

  const loadUserData = async (uid) => {
    const { data: adminData } = await supabase.from("admin_users").select("id").eq("id", uid).single();
    if (adminData) { setIsAdmin(true); setLoading(false); return; }
    const { data: profileData } = await supabase.from("profiles").select("*").eq("id", uid).single();
    setProfile(profileData);
    setLoading(false);
  };

  const loadMenu = async () => {
    const { data: items } = await supabase.from("menu_items").select("*, categories(name)").order("position");
    if (items) setMenu(items.map(i => ({
      id: i.id, category: i.categories?.name, name: i.name,
      desc: i.description, price: i.price, active: i.is_active,
      customizable: i.is_customizable,
    })));
  };

  const loadIngredients = async () => {
    const { data } = await supabase.from("sandwich_ingredients").select("*").eq("is_active", true).order("position");
    if (data) setIngredients({
      breads:   data.filter(i => i.type === "bread").map(i => i.name),
      proteins: data.filter(i => i.type === "protein").map(i => i.name),
      veggies:  data.filter(i => i.type === "veggie").map(i => i.name),
      sauces:   data.filter(i => i.type === "sauce").map(i => i.name),
    });
  };

  const loadStock = async () => {
    const { data } = await supabase.from("stock_items").select("*").order("name");
    if (data) {
      const itemsWithHistory = await Promise.all(data.map(async (item) => {
        const { data: moves } = await supabase.from("stock_movements")
          .select("*").eq("stock_item_id", item.id).order("created_at", { ascending: false }).limit(10);
        return {
          id: item.id, name: item.name, qty: item.quantity,
          min: item.min_alert, unit: item.unit,
          history: (moves || []).map(m => ({
            date: new Date(m.created_at).toLocaleDateString("fr-FR", { day:"2-digit", month:"2-digit" }),
            type: m.type, qty: m.quantity, note: m.note,
          })),
        };
      }));
      setStock(itemsWithHistory);
    }
  };

  const loadOrders = async (uid, adminMode = false) => {
    if (adminMode) {
      const { data } = await supabase.from("orders_with_profile").select("*, order_items(*)").order("created_at", { ascending: false });
      if (data) setOrders(data.map(mapOrder));
    } else {
      const { data } = await supabase.from("orders").select("*, order_items(*)").eq("user_id", uid).order("created_at", { ascending: false });
      if (data) setOrders(data.map(mapOrder));
    }
  };

  const mapOrder = (o) => ({
    id: o.id, status: o.status, position: o.queue_position,
    time: new Date(o.created_at).toLocaleTimeString("fr-FR", { hour:"2-digit", minute:"2-digit" }),
    total: o.total,
    user: { card: o.card_number, firstName: o.first_name, lastName: o.last_name },
    items: (o.order_items || []).map(i => ({
      id: i.id, name: i.menu_item_name, price: i.unit_price, qty: i.quantity,
      desc: i.customization ? `${i.customization.bread||""} ${i.customization.protein||""}` : "",
    })),
  });

  useEffect(() => {
    loadMenu(); loadIngredients();
    const menuSub = supabase.channel("menu_realtime")
      .on("postgres_changes", { event: "*", schema: "public", table: "menu_items" }, loadMenu).subscribe();
    return () => supabase.removeChannel(menuSub);
  }, []);

  useEffect(() => {
    if (!session) return;
    loadOrders(session.user.id, isAdmin);
    if (isAdmin) loadStock();
    const orderSub = supabase.channel("orders_realtime")
      .on("postgres_changes", { event: "*", schema: "public", table: "orders" }, () => loadOrders(session.user.id, isAdmin)).subscribe();
    return () => supabase.removeChannel(orderSub);
  }, [session, isAdmin]);

  const signIn = async (cardNumber, password) => {
    const { data: profileData } = await supabase.from("profiles").select("id").eq("card_number", cardNumber).single();
    if (!profileData) return { error: "Numéro de carte introuvable." };
    const { error } = await supabase.auth.signInWithPassword({ email: `${cardNumber}@ubreak.dz`, password });
    if (error) return { error: "Mot de passe incorrect." };
    return { ok: true };
  };

  const signInAdmin = async (email, password) => {
    const { error } = await supabase.auth.signInWithPassword({ email, password });
    if (error) return { error: "Identifiants incorrects." };
    return { ok: true };
  };

  const signUp = async (cardNumber, firstName, lastName, password) => {
    const { data, error } = await supabase.auth.signUp({ email: `${cardNumber}@ubreak.dz`, password });
    if (error) return { error: error.message };
    if (data.user) {
      const { error: profileError } = await supabase.from("profiles").insert({
        id: data.user.id, card_number: cardNumber, first_name: firstName, last_name: lastName,
      });
      if (profileError) return { error: profileError.message };
    }
    return { ok: true };
  };

  const signOut = async () => {
    await supabase.auth.signOut();
    setProfile(null); setIsAdmin(false); setOrders([]);
  };

  const placeOrder = async (cartItems) => {
    const total = cartItems.reduce((a, b) => a + b.qty * b.price, 0);
    const { data: orderData, error } = await supabase.from("orders").insert({ user_id: session.user.id, status: 0, total }).select().single();
    if (error || !orderData) return { error: "Erreur lors de la commande." };
    const orderItems = cartItems.map(item => ({
      order_id: orderData.id, menu_item_id: item.originalId || null,
      menu_item_name: item.name, unit_price: item.price,
      quantity: item.qty, customization: item.customization || null,
    }));
    await supabase.from("order_items").insert(orderItems);
    await loadOrders(session.user.id, false);
    return { id: orderData.id, position: orderData.queue_position };
  };

  const updateOrderStatus = async (id, status) => { await supabase.from("orders").update({ status }).eq("id", id); };
  const updateMenuItem = async (item) => { await supabase.from("menu_items").update({ name: item.name, description: item.desc, price: item.price, is_active: item.active, is_customizable: item.customizable }).eq("id", item.id); };
  const addMenuItem = async (item) => { const { data: cat } = await supabase.from("categories").select("id").eq("name", item.category).single(); await supabase.from("menu_items").insert({ category_id: cat?.id, name: item.name, description: item.desc, price: item.price, is_active: item.active, is_customizable: item.customizable }); };
  const deleteMenuItem = async (id) => { await supabase.from("menu_items").delete().eq("id", id); };
  const toggleMenuItem = async (id, active) => { await supabase.from("menu_items").update({ is_active: active }).eq("id", id); };
  const addIngredient = async (type, name) => { await supabase.from("sandwich_ingredients").insert({ type, name, is_active: true }); await loadIngredients(); };
  const removeIngredient = async (type, name) => { await supabase.from("sandwich_ingredients").update({ is_active: false }).eq("type", type).eq("name", name); await loadIngredients(); };
  const addStockItem = async (item) => { await supabase.from("stock_items").insert({ name: item.name, quantity: item.qty, min_alert: item.min, unit: item.unit }); await loadStock(); };
  const updateStockItem = async (item) => { await supabase.from("stock_items").update({ min_alert: item.min, unit: item.unit }).eq("id", item.id); await loadStock(); };
  const applyStockMovement = async (itemId, type, qty, note) => {
    const item = stock.find(s => s.id === itemId);
    if (!item) return;
    const newQty = Math.max(0, item.qty + (type === "in" ? qty : -qty));
    await supabase.from("stock_items").update({ quantity: newQty }).eq("id", itemId);
    await supabase.from("stock_movements").insert({ stock_item_id: itemId, type, quantity: qty, note });
    await loadStock();
  };

  return (
    <StoreCtx.Provider value={{
      session, profile, isAdmin, menu, ingredients, stock, orders, loading,
      signIn, signInAdmin, signUp, signOut, placeOrder, updateOrderStatus,
      updateMenuItem, addMenuItem, deleteMenuItem, toggleMenuItem,
      addIngredient, removeIngredient, addStockItem, updateStockItem, applyStockMovement,
      reloadStock: loadStock, reloadOrders: () => loadOrders(session?.user?.id, isAdmin),
    }}>
      {children}
    </StoreCtx.Provider>
  );
}
function Btn({ children, onClick, color=C.green, outline=false, small=false, danger=false, full=false, disabled=false, loading=false }) {
  const bg = danger ? C.danger : color;
  return (
    <button onClick={onClick} disabled={disabled||loading} style={{
      background:outline?"transparent":(disabled||loading?"#ccc":bg),
      color:outline?bg:"#fff", border:`2px solid ${disabled||loading?"#ccc":bg}`,
      borderRadius:10, padding:small?"6px 14px":"12px 22px",
      fontWeight:800, fontSize:small?12:14, cursor:disabled||loading?"not-allowed":"pointer",
      fontFamily:"inherit", width:full?"100%":"auto", opacity:disabled||loading?0.6:1,
    }}>{loading?"⏳ ...":children}</button>
  );
}

function Input({ label, value, onChange, type="text", placeholder }) {
  return (
    <div style={{marginBottom:12}}>
      {label&&<div style={{fontSize:11,fontWeight:700,color:C.textMuted,marginBottom:5,textTransform:"uppercase",letterSpacing:.7}}>{label}</div>}
      <input type={type} value={value} onChange={e=>onChange(e.target.value)} placeholder={placeholder}
        style={{width:"100%",padding:"11px 14px",border:`2px solid ${C.border}`,borderRadius:10,fontSize:14,fontFamily:"inherit",color:C.text,background:C.white,outline:"none",boxSizing:"border-box"}}
        onFocus={e=>e.target.style.borderColor=C.green} onBlur={e=>e.target.style.borderColor=C.border}
      />
    </div>
  );
}

function Toggle({ value, onChange }) {
  return (
    <div onClick={()=>onChange(!value)} style={{width:44,height:24,borderRadius:12,background:value?C.green:"#CBD5C0",position:"relative",cursor:"pointer",transition:"background .25s",flexShrink:0}}>
      <div style={{position:"absolute",top:3,left:value?23:3,width:18,height:18,borderRadius:"50%",background:"#fff",boxShadow:"0 1px 4px rgba(0,0,0,.2)",transition:"left .25s"}}/>
    </div>
  );
}

function Modal({ title, children, onClose }) {
  return (
    <div style={{position:"fixed",inset:0,background:"rgba(0,0,0,.45)",zIndex:200,display:"flex",alignItems:"center",justifyContent:"center",padding:20}} onClick={e=>{if(e.target===e.currentTarget)onClose();}}>
      <div style={{background:C.white,borderRadius:20,padding:28,width:"100%",maxWidth:520,maxHeight:"90vh",overflowY:"auto",boxShadow:"0 20px 60px rgba(0,0,0,.2)"}}>
        <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:20}}>
          <div style={{fontWeight:900,fontSize:18,color:C.text}}>{title}</div>
          <div onClick={onClose} style={{cursor:"pointer",fontSize:22,color:C.textMuted}}>✕</div>
        </div>
        {children}
      </div>
    </div>
  );
}

function Select({ label, value, onChange, options }) {
  return (
    <div style={{marginBottom:12}}>
      {label&&<div style={{fontSize:11,fontWeight:700,color:C.textMuted,marginBottom:5,textTransform:"uppercase",letterSpacing:.7}}>{label}</div>}
      <select value={value} onChange={e=>onChange(e.target.value)}
        style={{width:"100%",padding:"11px 14px",border:`2px solid ${C.border}`,borderRadius:10,fontSize:14,fontFamily:"inherit",color:C.text,background:C.white,outline:"none",boxSizing:"border-box"}}
        onFocus={e=>e.target.style.borderColor=C.green} onBlur={e=>e.target.style.borderColor=C.border}
      >{options.map(o=><option key={o} value={o}>{o}</option>)}</select>
    </div>
  );
}

function Badge({ children, color=C.green, bg }) {
  return <span style={{background:bg||color+"18",color,fontWeight:700,fontSize:11,padding:"3px 10px",borderRadius:20}}>{children}</span>;
}

function SplashScreen({ onDone }) {
  useEffect(()=>{ const t=setTimeout(onDone,2000); return ()=>clearTimeout(t); },[]);
  return (
    <div style={{flex:1,background:`linear-gradient(160deg,${C.greenDark},${C.green} 60%,${C.greenLight})`,display:"flex",flexDirection:"column",alignItems:"center",justifyContent:"center",gap:20,minHeight:"100vh"}}>
      <div style={{fontSize:80}}>🍽️</div>
      <div style={{color:"#fff",fontSize:36,fontWeight:900,letterSpacing:-1}}>U'BREAK</div>
      <div style={{color:"rgba(255,255,255,.75)",fontSize:14,fontStyle:"italic"}}>Fresh Bite, Fresh Mind</div>
    </div>
  );
}

function LoginScreen({ onGoRegister, onAdminRequest }) {
  const { signIn } = useStore();
  const [card,setCard]=useState(""); const [pwd,setPwd]=useState(""); const [err,setErr]=useState(""); const [busy,setBusy]=useState(false);
  const handle=async()=>{
    if(!card||!pwd){setErr("Remplissez tous les champs.");return;}
    setBusy(true); setErr("");
    const res = await signIn(card, pwd);
    if(res.error){setErr(res.error); setBusy(false);}
  };
  return (
    <div style={{flex:1,display:"flex",flexDirection:"column"}}>
      <div style={{background:`linear-gradient(160deg,${C.greenDark},${C.green})`,padding:"40px 28px 32px",borderBottomLeftRadius:32,borderBottomRightRadius:32}}>
        <div style={{display:"flex",alignItems:"center",gap:10}}>
          <div style={{width:44,height:44,background:C.green,borderRadius:"50%",display:"flex",alignItems:"center",justifyContent:"center",fontSize:22}}>🍽️</div>
          <div><div style={{fontWeight:900,fontSize:20,color:"#fff"}}>U'BREAK</div><div style={{fontSize:12,color:"rgba(255,255,255,.7)",fontStyle:"italic"}}>Fresh Bite, Fresh Mind</div></div>
        </div>
        <div style={{color:"rgba(255,255,255,.85)",marginTop:16,fontSize:15}}>Content de te revoir ! 👋</div>
      </div>
      <div style={{flex:1,padding:"28px 24px",overflowY:"auto"}}>
        <div style={{fontSize:22,fontWeight:900,color:C.text,marginBottom:24}}>Connexion</div>
        <Input label="N° Carte étudiante" value={card} onChange={setCard} placeholder="Ex: 20241234"/>
        <Input label="Mot de passe" type="password" value={pwd} onChange={setPwd} placeholder="••••••••"/>
        {err&&<div style={{color:C.danger,fontSize:13,marginBottom:10,fontWeight:600}}>⚠️ {err}</div>}
        <Btn full onClick={handle} loading={busy}>Se connecter</Btn>
        <div style={{textAlign:"center",marginTop:20,color:C.textMuted,fontSize:14}}>
          Pas encore de compte ?{" "}<span onClick={onGoRegister} style={{color:C.green,fontWeight:800,cursor:"pointer"}}>S'inscrire</span>
        </div>
      </div>
      <div onClick={onAdminRequest} style={{position:"fixed",bottom:16,right:16,background:C.greenDark,color:"#fff",borderRadius:20,padding:"6px 14px",fontSize:11,fontWeight:800,cursor:"pointer",opacity:.7}}>⚙️ Admin</div>
    </div>
  );
}

function RegisterScreen({ onGoLogin }) {
  const { signUp } = useStore();
  const [form,setForm]=useState({card:"",firstName:"",lastName:"",pwd:"",pwd2:""});
  const [err,setErr]=useState(""); const [busy,setBusy]=useState(false);
  const set=k=>v=>setForm(f=>({...f,[k]:v}));
  const handle=async()=>{
    if(!form.card||!form.firstName||!form.lastName||!form.pwd){setErr("Tous les champs sont obligatoires.");return;}
    if(form.pwd!==form.pwd2){setErr("Les mots de passe ne correspondent pas.");return;}
    setBusy(true); setErr("");
    const res = await signUp(form.card, form.firstName, form.lastName, form.pwd);
    if(res.error){setErr(res.error); setBusy(false);}
  };
  return (
    <div style={{flex:1,display:"flex",flexDirection:"column"}}>
      <div style={{background:`linear-gradient(160deg,${C.greenDark},${C.green})`,padding:"40px 28px 32px",borderBottomLeftRadius:32,borderBottomRightRadius:32}}>
        <div style={{display:"flex",alignItems:"center",gap:10}}>
          <div style={{width:44,height:44,background:C.green,borderRadius:"50%",display:"flex",alignItems:"center",justifyContent:"center",fontSize:22}}>🍽️</div>
          <div><div style={{fontWeight:900,fontSize:20,color:"#fff"}}>U'BREAK</div></div>
        </div>
        <div style={{color:"rgba(255,255,255,.85)",marginTop:16,fontSize:15}}>Rejoins la communauté U'Break 🎓</div>
      </div>
      <div style={{flex:1,padding:"28px 24px",overflowY:"auto"}}>
        <div style={{fontSize:22,fontWeight:900,color:C.text,marginBottom:24}}>Créer un compte</div>
        <Input label="N° Carte étudiante" value={form.card} onChange={set("card")} placeholder="Ex: 20241234"/>
        <Input label="Prénom" value={form.firstName} onChange={set("firstName")} placeholder="Mohamed"/>
        <Input label="Nom" value={form.lastName} onChange={set("lastName")} placeholder="Benali"/>
        <Input label="Mot de passe" type="password" value={form.pwd} onChange={set("pwd")} placeholder="••••••••"/>
        <Input label="Confirmer le mot de passe" type="password" value={form.pwd2} onChange={set("pwd2")} placeholder="••••••••"/>
        {err&&<div style={{color:C.danger,fontSize:13,marginBottom:10,fontWeight:600}}>⚠️ {err}</div>}
        <Btn full onClick={handle} loading={busy}>Créer mon compte</Btn>
        <div style={{textAlign:"center",marginTop:20,color:C.textMuted,fontSize:14}}>
          Déjà inscrit ?{" "}<span onClick={onGoLogin} style={{color:C.green,fontWeight:800,cursor:"pointer"}}>Se connecter</span>
        </div>
      </div>
    </div>
  );
}

function MenuScreen({ cart, setCart, onCheckout, onLogout }) {
  const { menu, ingredients, profile } = useStore();
  const activeMenu = menu.filter(i => i.active);
  const grouped = CATEGORIES.reduce((acc, cat) => { acc[cat]=activeMenu.filter(i=>i.category===cat); return acc; }, {});
  const availableCats = CATEGORIES.filter(c => grouped[c].length > 0);
  const [activeCat,setActiveCat]=useState(availableCats[0]||"");
  const [customizing,setCustomizing]=useState(null);
  const [sandwich,setSandwich]=useState({bread:"",protein:"",veggies:[],sauce:""});
  const cartCount=Object.values(cart).reduce((a,b)=>a+b.qty,0);
  const cartTotal=Object.values(cart).reduce((a,b)=>a+b.qty*b.price,0);

  const addItem=(item)=>{
    if(item.customizable){ setCustomizing(item); setSandwich({bread:ingredients.breads[0]||"",protein:ingredients.proteins[0]||"",veggies:[],sauce:ingredients.sauces[0]||""}); return; }
    setCart(prev=>({...prev,[item.id]:{...item,qty:(prev[item.id]?.qty||0)+1}}));
  };
  const confirmSandwich=()=>{
    const key=`${customizing.id}-${Date.now()}`;
    setCart(prev=>({...prev,[key]:{...customizing,id:key,originalId:customizing.id,desc:`${sandwich.bread}, ${sandwich.protein}, ${sandwich.veggies.join("/")||"sans légume"}, ${sandwich.sauce}`,qty:1,customization:sandwich}}));
    setCustomizing(null);
  };

  if(customizing) return (
    <div style={{flex:1,display:"flex",flexDirection:"column"}}>
      <div style={{background:C.green,padding:"40px 24px 20px",color:"#fff"}}>
        <div style={{display:"flex",alignItems:"center",gap:12}}>
          <span onClick={()=>setCustomizing(null)} style={{fontSize:22,cursor:"pointer"}}>←</span>
          <div><div style={{fontWeight:900,fontSize:18}}>Personnalise ton sandwich</div><div style={{fontSize:13,opacity:.8}}>{customizing.name}</div></div>
        </div>
      </div>
      <div style={{flex:1,overflowY:"auto",padding:"20px 20px 100px"}}>
        {[["🍞 Pain","breads","bread"],["🥩 Protéine","proteins","protein"],["🥫 Sauce","sauces","sauce"]].map(([label,key,field])=>(
          <div key={field}>
            <div style={{fontWeight:800,fontSize:13,color:C.textMuted,textTransform:"uppercase",letterSpacing:.8,marginBottom:10,marginTop:6}}>{label}</div>
            <div style={{display:"flex",flexWrap:"wrap",gap:8,marginBottom:16}}>
              {ingredients[key].map(o=><div key={o} onClick={()=>setSandwich(s=>({...s,[field]:o}))} style={{padding:"8px 14px",borderRadius:20,fontSize:13,fontWeight:700,cursor:"pointer",background:sandwich[field]===o?C.green:C.border,color:sandwich[field]===o?"#fff":C.text}}>{o}</div>)}
            </div>
          </div>
        ))}
        <div style={{fontWeight:800,fontSize:13,color:C.textMuted,textTransform:"uppercase",letterSpacing:.8,marginBottom:10,marginTop:6}}>🥦 Légumes</div>
        <div style={{display:"flex",flexWrap:"wrap",gap:8,marginBottom:16}}>
          {ingredients.veggies.map(v=><div key={v} onClick={()=>setSandwich(s=>({...s,veggies:s.veggies.includes(v)?s.veggies.filter(x=>x!==v):[...s.veggies,v]}))} style={{padding:"8px 14px",borderRadius:20,fontSize:13,fontWeight:700,cursor:"pointer",background:sandwich.veggies.includes(v)?C.green:C.border,color:sandwich.veggies.includes(v)?"#fff":C.text}}>{v}</div>)}
        </div>
      </div>
      <div style={{position:"fixed",bottom:0,left:0,right:0,padding:"16px 20px",background:C.white,borderTop:`1px solid ${C.border}`}}>
        <Btn full onClick={confirmSandwich}>Ajouter — {customizing.price} DA</Btn>
      </div>
    </div>
  );

  return (
    <div style={{flex:1,display:"flex",flexDirection:"column"}}>
      <div style={{background:C.green,padding:"36px 24px 16px"}}>
        <div style={{display:"flex",justifyContent:"space-between",alignItems:"center"}}>
          <div>
            <div style={{color:"rgba(255,255,255,.75)",fontSize:13}}>Bonjour 👋</div>
            <div style={{color:"#fff",fontWeight:900,fontSize:18}}>{profile?.first_name} {profile?.last_name}</div>
          </div>
          <div style={{display:"flex",gap:10,alignItems:"center"}}>
            <div style={{background:"rgba(255,255,255,.2)",padding:"6px 12px",borderRadius:20,color:"#fff",fontSize:12,fontWeight:700}}>🎓 {profile?.card_number}</div>
            <div onClick={onLogout} style={{background:"rgba(255,255,255,.15)",borderRadius:20,padding:"6px 10px",cursor:"pointer",fontSize:12,color:"#fff",fontWeight:700}}>↩</div>
          </div>
        </div>
      </div>
      <div style={{background:C.white,borderBottom:`1px solid ${C.border}`,overflowX:"auto",display:"flex",padding:"0 8px"}}>
        {availableCats.map(cat=>(
          <div key={cat} onClick={()=>setActiveCat(cat)} style={{padding:"12px 14px",cursor:"pointer",whiteSpace:"nowrap",fontSize:13,fontWeight:800,color:activeCat===cat?C.green:C.textMuted,borderBottom:activeCat===cat?`3px solid ${C.green}`:"3px solid transparent"}}>{cat}</div>
        ))}
      </div>
      <div style={{flex:1,overflowY:"auto",padding:"16px 16px 120px"}}>
        {(grouped[activeCat]||[]).map(item=>(
          <div key={item.id} style={{background:C.white,borderRadius:16,padding:"14px 16px",marginBottom:12,display:"flex",justifyContent:"space-between",alignItems:"center",boxShadow:"0 2px 10px rgba(0,0,0,.06)",border:`1px solid ${C.border}`}}>
            <div style={{flex:1}}>
              <div style={{fontWeight:800,fontSize:15,color:C.text}}>{item.name}</div>
              <div style={{fontSize:12,color:C.textMuted,marginTop:2}}>{item.desc}</div>
              {item.customizable&&<div style={{fontSize:11,color:C.green,fontWeight:700,marginTop:4}}>✏️ Personnalisable</div>}
            </div>
            <div style={{display:"flex",alignItems:"center",gap:10,marginLeft:10}}>
              <div style={{fontWeight:900,color:C.green,fontSize:15}}>{item.price} DA</div>
              <div onClick={()=>addItem(item)} style={{width:34,height:34,borderRadius:"50%",background:C.green,color:"#fff",display:"flex",alignItems:"center",justifyContent:"center",fontSize:22,cursor:"pointer",fontWeight:900}}>+</div>
            </div>
          </div>
        ))}
      </div>
      {cartCount>0&&(
        <div onClick={onCheckout} style={{position:"fixed",bottom:20,left:16,right:16,background:C.green,borderRadius:18,padding:"16px 20px",display:"flex",justifyContent:"space-between",alignItems:"center",boxShadow:"0 8px 24px rgba(78,154,14,.45)",cursor:"pointer"}}>
          <div style={{background:"rgba(255,255,255,.25)",borderRadius:10,width:28,height:28,display:"flex",alignItems:"center",justifyContent:"center",color:"#fff",fontWeight:900}}>{cartCount}</div>
          <div style={{color:"#fff",fontWeight:900,fontSize:15}}>Voir mon panier</div>
          <div style={{color:"#fff",fontWeight:900}}>{cartTotal} DA</div>
        </div>
      )}
    </div>
  );
}

function CartScreen({ cart, setCart, onOrder, onBack }) {
  const [busy,setBusy]=useState(false);
  const items=Object.values(cart).filter(i=>i.qty>0);
  const total=items.reduce((a,b)=>a+b.qty*b.price,0);
  const change=(id,delta)=>setCart(prev=>{ const u={...prev,[id]:{...prev[id],qty:(prev[id]?.qty||0)+delta}}; if(u[id].qty<=0){const{[id]:_,...rest}=u;return rest;} return u; });
  return (
    <div style={{flex:1,display:"flex",flexDirection:"column"}}>
      <div style={{background:C.green,padding:"40px 24px 20px",color:"#fff"}}>
        <div style={{display:"flex",alignItems:"center",gap:12}}>
          <span onClick={onBack} style={{fontSize:22,cursor:"pointer"}}>←</span>
          <div style={{fontWeight:900,fontSize:20}}>Mon panier 🛒</div>
        </div>
      </div>
      <div style={{flex:1,overflowY:"auto",padding:"16px 16px 120px"}}>
        {items.length===0&&<div style={{textAlign:"center",color:C.textMuted,marginTop:60,fontSize:16}}>Ton panier est vide 😅</div>}
        {items.map(item=>(
          <div key={item.id} style={{background:C.white,borderRadius:16,padding:"14px 16px",marginBottom:10,display:"flex",alignItems:"center",justifyContent:"space-between",border:`1px solid ${C.border}`}}>
            <div style={{flex:1}}>
              <div style={{fontWeight:800,fontSize:14}}>{item.name}</div>
              <div style={{fontSize:11,color:C.textMuted,marginTop:2}}>{item.desc}</div>
              <div style={{color:C.green,fontWeight:900,fontSize:14,marginTop:4}}>{item.qty*item.price} DA</div>
            </div>
            <div style={{display:"flex",alignItems:"center",gap:10}}>
              <div onClick={()=>change(item.id,-1)} style={{width:28,height:28,borderRadius:"50%",background:C.border,display:"flex",alignItems:"center",justifyContent:"center",cursor:"pointer",fontWeight:900,fontSize:16}}>−</div>
              <span style={{fontWeight:900,fontSize:16,minWidth:16,textAlign:"center"}}>{item.qty}</span>
              <div onClick={()=>change(item.id,1)} style={{width:28,height:28,borderRadius:"50%",background:C.green,color:"#fff",display:"flex",alignItems:"center",justifyContent:"center",cursor:"pointer",fontWeight:900,fontSize:18}}>+</div>
            </div>
          </div>
        ))}
        {items.length>0&&<div style={{background:C.white,borderRadius:16,padding:16,border:`1px solid ${C.border}`}}><div style={{display:"flex",justifyContent:"space-between",fontWeight:800,fontSize:16}}><span>Total</span><span style={{color:C.green}}>{total} DA</span></div></div>}
      </div>
      {items.length>0&&(
        <div style={{position:"fixed",bottom:0,left:0,right:0,padding:"16px 20px",background:C.white,borderTop:`1px solid ${C.border}`}}>
          <Btn full onClick={async()=>{setBusy(true);await onOrder();setBusy(false);}} loading={busy}>Commander — {total} DA</Btn>
        </div>
      )}
    </div>
  );
}

function TrackingScreen({ orderId, onBack }) {
  const { orders } = useStore();
  const order = orders.find(o => o.id === orderId);
  const activeOrders = orders.filter(o => o.status < 2);
  const myPos = activeOrders.findIndex(o => o.id === orderId) + 1;
  const status = order?.status ?? 0;
  const STATUSES=[{label:"Commande reçue",icon:"✅",color:C.green},{label:"En préparation",icon:"👨‍🍳",color:C.warning},{label:"Prête à récupérer !",icon:"🎉",color:C.green}];
  return (
    <div style={{flex:1,display:"flex",flexDirection:"column"}}>
      <div style={{background:C.green,padding:"40px 24px 24px",color:"#fff"}}>
        <div style={{display:"flex",alignItems:"center",gap:12,marginBottom:16}}>
          <span onClick={onBack} style={{fontSize:22,cursor:"pointer"}}>←</span>
          <div style={{fontWeight:900,fontSize:20}}>Suivi de commande</div>
        </div>
        <div style={{background:"rgba(255,255,255,.15)",borderRadius:14,padding:"12px 16px"}}>
          <div style={{fontSize:12,opacity:.8,marginBottom:2}}>N° de commande</div>
          <div style={{fontWeight:900,fontSize:16}}>{orderId?.slice(0,8).toUpperCase()}...</div>
        </div>
      </div>
      <div style={{flex:1,overflowY:"auto",padding:"20px 20px 30px"}}>
        <div style={{background:status===2?"#E8F8DC":"#FFF3E0",border:`2px solid ${STATUSES[status].color}`,borderRadius:16,padding:"14px 18px",marginBottom:20,display:"flex",alignItems:"center",gap:12}}>
          <div style={{fontSize:32}}>{STATUSES[status].icon}</div>
          <div>
            <div style={{fontWeight:900,fontSize:16,color:STATUSES[status].color}}>{STATUSES[status].label}</div>
            <div style={{fontSize:12,color:C.textMuted,marginTop:2}}>Mis à jour en temps réel</div>
          </div>
        </div>
        {status<2&&myPos>0&&(
          <div style={{background:C.white,borderRadius:16,padding:18,marginBottom:16,border:`1px solid ${C.border}`}}>
            <div style={{fontWeight:800,fontSize:15,marginBottom:12}}>📋 File d'attente — Position #{myPos}</div>
            <div style={{display:"flex",gap:6,flexWrap:"wrap"}}>
              {activeOrders.map((o,i)=>(
                <div key={o.id} style={{minWidth:70,background:o.id===orderId?"#E8F8DC":C.cream,borderRadius:10,padding:"8px 10px",fontSize:11,borderLeft:`3px solid ${o.id===orderId?C.green:C.border}`}}>
                  <div style={{fontWeight:700,color:o.id===orderId?C.green:C.textMuted}}>{o.id===orderId?"✦ Toi":`Cmd #${i+1}`}</div>
                  <div style={{fontSize:10,color:C.textMuted,marginTop:2}}>{o.time}</div>
                </div>
              ))}
            </div>
          </div>
        )}
        {status===2&&(
          <div style={{background:`linear-gradient(135deg,${C.green},${C.greenLight})`,borderRadius:20,padding:24,textAlign:"center",color:"#fff",marginBottom:16}}>
            <div style={{fontSize:48,marginBottom:8}}>🎉</div>
            <div style={{fontWeight:900,fontSize:20}}>Ta commande est prête !</div>
            <div style={{opacity:.85,marginTop:6}}>Rendez-vous au comptoir U'Break</div>
          </div>
        )}
      </div>
    </div>
  );
}
function AdminLoginModal({ onCancel }) {
  const { signInAdmin } = useStore();
  const [email,setEmail]=useState(""); const [pwd,setPwd]=useState("");
  const [err,setErr]=useState(""); const [busy,setBusy]=useState(false);
  const [attempts,setAttempts]=useState(0); const [locked,setLocked]=useState(false); const [lockTimer,setLockTimer]=useState(0);
  useEffect(()=>{
    if(!locked)return;
    setLockTimer(30);
    const iv=setInterval(()=>setLockTimer(t=>{ if(t<=1){clearInterval(iv);setLocked(false);setAttempts(0);return 0;} return t-1; }),1000);
    return()=>clearInterval(iv);
  },[locked]);
  const handle=async()=>{
    if(locked||!email||!pwd)return;
    setBusy(true); setErr("");
    const res=await signInAdmin(email,pwd);
    if(res.error){ const na=attempts+1; setAttempts(na); setPwd(""); setBusy(false); if(na>=3){setLocked(true);}else{setErr(`Identifiants incorrects. ${3-na} tentative(s) restante(s).`);} }
  };
  return (
    <div style={{position:"fixed",inset:0,background:"rgba(0,0,0,.65)",zIndex:300,display:"flex",alignItems:"center",justifyContent:"center",padding:20}}>
      <div style={{background:C.white,borderRadius:24,padding:36,width:"100%",maxWidth:380,boxShadow:"0 24px 64px rgba(0,0,0,.3)"}}>
        <div style={{textAlign:"center",marginBottom:28}}>
          <div style={{width:60,height:60,background:C.greenDark,borderRadius:"50%",display:"flex",alignItems:"center",justifyContent:"center",fontSize:26,margin:"0 auto 14px"}}>🔒</div>
          <div style={{fontWeight:900,fontSize:22,color:C.text}}>Accès administrateur</div>
          <div style={{fontSize:13,color:C.textMuted,marginTop:6}}>Réservé au gérant U'Break</div>
        </div>
        {locked?(
          <div style={{background:C.dangerLight,border:`1px solid ${C.danger}40`,borderRadius:14,padding:"16px 20px",textAlign:"center",marginBottom:20}}>
            <div style={{fontSize:28,marginBottom:8}}>🚫</div>
            <div style={{fontWeight:800,color:C.danger,fontSize:15}}>Accès bloqué</div>
            <div style={{fontSize:13,color:C.textMuted,marginTop:4}}>Réessaie dans <strong>{lockTimer}s</strong>.</div>
          </div>
        ):(
          <>
            <Input label="Email admin" value={email} onChange={setEmail} placeholder="gerant@ubreak.dz"/>
            <Input label="Mot de passe" type="password" value={pwd} onChange={setPwd} placeholder="••••••••"/>
            {err&&<div style={{color:C.danger,fontSize:12,fontWeight:700,marginBottom:12}}>⚠️ {err}</div>}
          </>
        )}
        <div style={{display:"flex",gap:10,marginTop:8}}>
          <button onClick={onCancel} style={{flex:1,padding:12,border:`2px solid ${C.border}`,borderRadius:10,background:"transparent",fontWeight:800,fontSize:14,color:C.textMuted,cursor:"pointer",fontFamily:"inherit"}}>Annuler</button>
          <button onClick={handle} disabled={locked||!email||!pwd||busy} style={{flex:1,padding:12,border:"none",borderRadius:10,background:locked||!email||!pwd?"#ccc":C.greenDark,fontWeight:800,fontSize:14,color:"#fff",cursor:locked||!email||!pwd?"not-allowed":"pointer",fontFamily:"inherit"}}>{busy?"⏳ ...":"Connexion"}</button>
        </div>
      </div>
    </div>
  );
}

const ADMIN_NAV=[{id:"dashboard",icon:"📊",label:"Tableau de bord"},{id:"orders",icon:"🧾",label:"Commandes"},{id:"menu",icon:"🍽️",label:"Menu"},{id:"ingredients",icon:"🥬",label:"Ingrédients"},{id:"stock",icon:"📦",label:"Stocks"}];

function AdminApp({ onLogout }) {
  const { stock, orders, menu, signOut } = useStore();
  const [nav,setNav]=useState("dashboard");
  const lowCount=stock.filter(s=>s.qty<=s.min).length;
  const handleLogout=async()=>{ await signOut(); onLogout(); };
  const activeItems=menu.filter(i=>i.active).length;
  const pendingOrders=orders.filter(o=>o.status<2).length;

  const renderSection=()=>{
    if(nav==="orders") return <AdminOrders/>;
    if(nav==="menu") return <AdminMenu/>;
    if(nav==="ingredients") return <AdminIngredients/>;
    if(nav==="stock") return <AdminStock/>;
    return (
      <div>
        <h1 style={{fontWeight:900,fontSize:26,color:C.text,marginBottom:4}}>Tableau de bord</h1>
        <div style={{fontSize:14,color:C.textMuted,marginBottom:24}}>Vue d'ensemble en temps réel</div>
        <div style={{display:"grid",gridTemplateColumns:"repeat(2,1fr)",gap:14,marginBottom:24}}>
          {[
            {label:"Commandes en cours",value:pendingOrders,icon:"🧾",color:pendingOrders>0?C.warning:C.success,nav:"orders"},
            {label:"Plats actifs",value:activeItems,icon:"🍽️",color:C.green,nav:"menu"},
            {label:"Alertes stock",value:lowCount,icon:"⚠️",color:lowCount>0?C.danger:C.success,nav:"stock"},
            {label:"Commandes totales",value:orders.length,icon:"📈",color:C.greenDark},
          ].map(kpi=>(
            <div key={kpi.label} onClick={()=>kpi.nav&&setNav(kpi.nav)} style={{background:C.white,borderRadius:16,padding:"18px 16px",border:`1px solid ${C.border}`,cursor:kpi.nav?"pointer":"default"}}>
              <div style={{fontSize:28,marginBottom:8}}>{kpi.icon}</div>
              <div style={{fontWeight:900,fontSize:28,color:kpi.color,lineHeight:1}}>{kpi.value}</div>
              <div style={{fontSize:12,color:C.textMuted,marginTop:4}}>{kpi.label}</div>
            </div>
          ))}
        </div>
        {lowCount>0&&(
          <div style={{background:C.dangerLight,border:`1px solid ${C.danger}40`,borderRadius:14,padding:"14px 18px"}}>
            <div style={{fontWeight:800,fontSize:14,color:C.danger,marginBottom:8}}>⚠️ Stocks bas</div>
            {stock.filter(s=>s.qty<=s.min).map(s=><div key={s.id} style={{fontSize:13,color:C.text,marginBottom:4}}>• {s.name} — {s.qty}/{s.min} {s.unit}</div>)}
          </div>
        )}
      </div>
    );
  };

  return (
    <div style={{display:"flex",flexDirection:"column",minHeight:"100vh",background:C.cream,fontFamily:"'Nunito','Segoe UI',sans-serif"}}>
      <div style={{background:C.sidebar,padding:"16px 20px",display:"flex",justifyContent:"space-between",alignItems:"center"}}>
        <div style={{display:"flex",alignItems:"center",gap:10}}>
          <div style={{width:34,height:34,background:C.green,borderRadius:"50%",display:"flex",alignItems:"center",justifyContent:"center",fontSize:16}}>🍽️</div>
          <div><div style={{fontWeight:900,fontSize:15,color:"#fff"}}>U'BREAK</div><div style={{fontSize:10,color:C.sidebarText,fontStyle:"italic"}}>Admin Panel</div></div>
        </div>
        <div style={{display:"flex",alignItems:"center",gap:10}}>
          {lowCount>0&&<div style={{background:C.danger,color:"#fff",fontWeight:800,fontSize:11,padding:"4px 10px",borderRadius:20}}>🔴 {lowCount}</div>}
          <div onClick={handleLogout} style={{background:"rgba(255,255,255,.1)",color:"#fff",fontWeight:800,fontSize:12,padding:"6px 12px",borderRadius:16,cursor:"pointer"}}>↩ Quitter</div>
        </div>
      </div>
      <div style={{flex:1,overflowY:"auto",padding:"20px 16px 100px"}}>
        {renderSection()}
      </div>
      <div style={{position:"fixed",bottom:0,left:0,right:0,background:C.sidebar,display:"flex",justifyContent:"space-around",padding:"8px 0 12px"}}>
        {ADMIN_NAV.map(item=>(
          <div key={item.id} onClick={()=>setNav(item.id)} style={{display:"flex",flexDirection:"column",alignItems:"center",gap:3,cursor:"pointer",opacity:nav===item.id?1:.5}}>
            <span style={{fontSize:20}}>{item.icon}</span>
            <span style={{fontSize:9,fontWeight:700,color:"#fff"}}>{item.label}</span>
          </div>
        ))}
      </div>
    </div>
  );
}

function AdminOrders() {
  const { orders, updateOrderStatus, reloadOrders } = useStore();
  const statusLabel=["Reçue","En préparation","Prête ✅"];
  const statusColor=[C.warning,C.warning,C.success];
  return (
    <div>
      <h1 style={{fontWeight:900,fontSize:22,color:C.text,marginBottom:16}}>Commandes</h1>
      {orders.length===0&&<div style={{textAlign:"center",color:C.textMuted,marginTop:40}}>Aucune commande pour l'instant.</div>}
      {[...orders].reverse().map(order=>(
        <div key={order.id} style={{background:C.white,borderRadius:16,padding:16,marginBottom:12,border:`1px solid ${C.border}`}}>
          <div style={{display:"flex",justifyContent:"space-between",alignItems:"flex-start",marginBottom:8}}>
            <div><div style={{fontWeight:900,fontSize:15}}>{order.id.slice(0,8).toUpperCase()}...</div><div style={{fontSize:12,color:C.textMuted}}>🎓 {order.user?.card} · {order.time}</div></div>
            <div style={{background:statusColor[order.status]+"22",color:statusColor[order.status],fontWeight:800,fontSize:11,padding:"4px 10px",borderRadius:20}}>{statusLabel[order.status]}</div>
          </div>
          <div style={{fontSize:13,color:C.text,marginBottom:10}}>{order.items.map(i=>`${i.name} ×${i.qty}`).join(" · ")}</div>
          <div style={{display:"flex",justifyContent:"space-between",alignItems:"center"}}>
            <div style={{fontWeight:900,color:C.green}}>{order.total} DA</div>
            {order.status<2&&<Btn small onClick={async()=>{ await updateOrderStatus(order.id,order.status+1); await reloadOrders(); }}>{order.status===0?"▶ Démarrer":"✅ Prête"}</Btn>}
          </div>
        </div>
      ))}
    </div>
  );
}

function AdminMenu() {
  const { menu, addMenuItem, updateMenuItem, deleteMenuItem, toggleMenuItem } = useStore();
  const [modal,setModal]=useState(null); const [busy,setBusy]=useState(false);
  const BLANK={id:"",category:CATEGORIES[0],name:"",price:"",desc:"",active:true,customizable:false};
  const [form,setForm]=useState(BLANK); const setF=k=>v=>setForm(f=>({...f,[k]:v}));
  const save=async()=>{ if(!form.name||!form.price)return; setBusy(true); if(modal==="add") await addMenuItem({...form,price:Number(form.price)}); else await updateMenuItem({...form,price:Number(form.price)}); setBusy(false); setModal(null); };
  return (
    <div>
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:16}}>
        <h1 style={{fontWeight:900,fontSize:22,color:C.text}}>Menu</h1>
        <Btn small onClick={()=>{setForm({...BLANK,id:"new"});setModal("add");}}>+ Nouveau</Btn>
      </div>
      {menu.map((item,idx)=>(
        <div key={item.id} style={{background:C.white,borderRadius:14,padding:"12px 14px",marginBottom:10,border:`1px solid ${C.border}`,display:"flex",justifyContent:"space-between",alignItems:"center"}}>
          <div style={{flex:1}}>
            <div style={{fontWeight:800,fontSize:14}}>{item.name}</div>
            <div style={{fontSize:11,color:C.textMuted}}>{item.category} · {item.price} DA</div>
          </div>
          <div style={{display:"flex",alignItems:"center",gap:10}}>
            <Toggle value={item.active} onChange={()=>toggleMenuItem(item.id,!item.active)}/>
            <Btn small outline onClick={()=>{setForm({...item,price:String(item.price)});setModal(item);}}>✏️</Btn>
            <Btn small danger onClick={()=>deleteMenuItem(item.id)}>🗑</Btn>
          </div>
        </div>
      ))}
      {modal!==null&&(
        <Modal title={modal==="add"?"Ajouter":"Modifier"} onClose={()=>setModal(null)}>
          <Select label="Catégorie" value={form.category} onChange={setF("category")} options={CATEGORIES}/>
          <Input label="Nom" value={form.name} onChange={setF("name")} placeholder="Ex: Box Riz Végétarien"/>
          <Input label="Description" value={form.desc} onChange={setF("desc")} placeholder="Ingrédients..."/>
          <Input label="Prix (DA)" type="number" value={form.price} onChange={setF("price")} placeholder="380"/>
          <div style={{display:"flex",gap:20,marginBottom:16}}>
            <label style={{display:"flex",alignItems:"center",gap:8,cursor:"pointer",fontSize:14,fontWeight:700}}><Toggle value={form.active} onChange={v=>setF("active")(v)}/>Actif</label>
            <label style={{display:"flex",alignItems:"center",gap:8,cursor:"pointer",fontSize:14,fontWeight:700}}><Toggle value={form.customizable} onChange={v=>setF("customizable")(v)}/>Perso.</label>
          </div>
          <div style={{display:"flex",gap:10,justifyContent:"flex-end"}}>
            <Btn outline onClick={()=>setModal(null)}>Annuler</Btn>
            <Btn onClick={save} disabled={!form.name||!form.price} loading={busy}>💾 Enregistrer</Btn>
          </div>
        </Modal>
      )}
    </div>
  );
}

function AdminIngredients() {
  const { ingredients, addIngredient, removeIngredient } = useStore();
  const [adding,setAdding]=useState({breads:"",proteins:"",veggies:"",sauces:""});
  const typeMap={breads:"bread",proteins:"protein",veggies:"veggie",sauces:"sauce"};
  return (
    <div>
      <h1 style={{fontWeight:900,fontSize:22,color:C.text,marginBottom:16}}>Ingrédients</h1>
      {[["🍞 Pain","breads"],["🥩 Protéines","proteins"],["🥦 Légumes","veggies"],["🥫 Sauces","sauces"]].map(([title,key])=>(
        <div key={key} style={{background:C.white,borderRadius:14,padding:16,border:`1px solid ${C.border}`,marginBottom:12}}>
          <div style={{fontWeight:800,fontSize:14,marginBottom:10}}>{title}</div>
          <div style={{display:"flex",flexWrap:"wrap",gap:8,marginBottom:10}}>
            {ingredients[key].map(item=>(
              <div key={item} style={{display:"flex",alignItems:"center",gap:6,background:C.greenGhost,border:`1px solid ${C.border}`,borderRadius:20,padding:"5px 12px"}}>
                <span style={{fontSize:13,fontWeight:700}}>{item}</span>
                <span onClick={()=>removeIngredient(typeMap[key],item)} style={{cursor:"pointer",color:C.danger,fontWeight:900,fontSize:16,lineHeight:1}}>×</span>
              </div>
            ))}
          </div>
          <div style={{display:"flex",gap:8}}>
            <input value={adding[key]} onChange={e=>setAdding(a=>({...a,[key]:e.target.value}))} placeholder="Ajouter..."
              onKeyDown={e=>{if(e.key==="Enter"&&adding[key].trim()){addIngredient(typeMap[key],adding[key].trim());setAdding(a=>({...a,[key]:""}));}}}
              style={{flex:1,padding:"8px 12px",border:`2px solid ${C.border}`,borderRadius:10,fontSize:13,fontFamily:"inherit",outline:"none"}}
            />
            <Btn small onClick={()=>{if(adding[key].trim()){addIngredient(typeMap[key],adding[key].trim());setAdding(a=>({...a,[key]:""}));}}}>+</Btn>
          </div>
        </div>
      ))}
    </div>
  );
}

function AdminStock() {
  const { stock, addStockItem, updateStockItem, applyStockMovement } = useStore();
  const [modal,setModal]=useState(null); const [busy,setBusy]=useState(false);
  const [form,setForm]=useState({id:"",name:"",qty:"",min:"",unit:"pcs"}); const setF=k=>v=>setForm(f=>({...f,[k]:v}));
  const [mv,setMv]=useState({type:"in",qty:"",note:""}); const [mvItem,setMvItem]=useState(null);
  const save=async()=>{ if(!form.min)return; setBusy(true); if(modal==="add") await addStockItem({name:form.name,qty:Number(form.qty),min:Number(form.min),unit:form.unit}); else await updateStockItem({id:form.id,min:Number(form.min),unit:form.unit}); setBusy(false); setModal(null); };
  return (
    <div>
      <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:16}}>
        <h1 style={{fontWeight:900,fontSize:22,color:C.text}}>Stocks</h1>
        <Btn small onClick={()=>{setForm({id:"",name:"",qty:"",min:"",unit:"pcs"});setModal("add");}}>+ Nouvel article</Btn>
      </div>
      {stock.map(item=>{ const isLow=item.qty<=item.min; return (
        <div key={item.id} style={{background:C.white,borderRadius:14,padding:16,marginBottom:12,border:`2px solid ${isLow?C.danger+"60":C.border}`}}>
          <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:8}}>
            <div><div style={{fontWeight:800,fontSize:14}}>{item.name}</div><div style={{fontSize:11,color:C.textMuted}}>Min: {item.min} {item.unit}</div></div>
            <div style={{fontWeight:900,fontSize:22,color:isLow?C.danger:C.text}}>{item.qty} <span style={{fontSize:12,fontWeight:600,color:C.textMuted}}>{item.unit}</span></div>
          </div>
          <div style={{display:"flex",gap:8}}>
            <Btn small outline onClick={()=>{setMvItem(item);setMv({type:"in",qty:"",note:""});}}>📋 Mouvement</Btn>
            <Btn small outline onClick={()=>{setForm({id:item.id,name:item.name,qty:String(item.qty),min:String(item.min),unit:item.unit});setModal(item);}}>⚙️</Btn>
          </div>
        </div>
      );})}
      {mvItem&&(
        <Modal title={`📦 ${mvItem.name}`} onClose={()=>setMvItem(null)}>
          <div style={{display:"flex",gap:8,marginBottom:12}}>
            {["in","out"].map(t=><div key={t} onClick={()=>setMv(m=>({...m,type:t}))} style={{flex:1,padding:10,borderRadius:12,textAlign:"center",cursor:"pointer",fontWeight:800,fontSize:14,background:mv.type===t?(t==="in"?C.successLight:C.dangerLight):C.cream,color:mv.type===t?(t==="in"?C.success:C.danger):C.textMuted,border:`2px solid ${mv.type===t?(t==="in"?C.success:C.danger):C.border}`}}>{t==="in"?"📥 Entrée":"📤 Sortie"}</div>)}
          </div>
          <Input label="Quantité" type="number" value={mv.qty} onChange={v=>setMv(m=>({...m,qty:v}))} placeholder="Ex: 10"/>
          <Input label="Note" value={mv.note} onChange={v=>setMv(m=>({...m,note:v}))} placeholder="Ex: Livraison"/>
          <Btn onClick={async()=>{if(!mv.qty)return;setBusy(true);await applyStockMovement(mvItem.id,mv.type,Number(mv.qty),mv.note);setBusy(false);setMvItem(null);}} disabled={!mv.qty} loading={busy}>✅ Valider</Btn>
        </Modal>
      )}
      {modal!==null&&(
        <Modal title={modal==="add"?"Nouvel article":"Modifier seuil"} onClose={()=>setModal(null)}>
          {modal==="add"&&<Input label="Nom" value={form.name} onChange={setF("name")} placeholder="Ex: Avocats"/>}
          {modal==="add"&&<Input label="Quantité actuelle" type="number" value={form.qty} onChange={setF("qty")} placeholder="12"/>}
          <Input label="Seuil d'alerte" type="number" value={form.min} onChange={setF("min")} placeholder="5"/>
          <Input label="Unité" value={form.unit} onChange={setF("unit")} placeholder="pcs, kg, litre..."/>
          <div style={{display:"flex",gap:10,justifyContent:"flex-end",marginTop:8}}>
            <Btn outline onClick={()=>setModal(null)}>Annuler</Btn>
            <Btn onClick={save} loading={busy}>💾 Enregistrer</Btn>
          </div>
        </Modal>
      )}
    </div>
  );
}

function ClientApp({ onAdminRequest }) {
  const { profile, placeOrder, signOut, loading } = useStore();
  const [screen,setScreen]=useState("splash");
  const [cart,setCart]=useState({});
  const [currentOrderId,setCurrentOrderId]=useState(null);
  useEffect(()=>{ if(profile && screen==="splash") setScreen("menu"); },[profile]);
  const doOrder=async()=>{ const res=await placeOrder(Object.values(cart)); if(res?.id){setCurrentOrderId(res.id);setCart({});setScreen("tracking");} };
  if(loading) return <div style={{display:"flex",alignItems:"center",justifyContent:"center",minHeight:"100vh",background:C.cream}}><div style={{fontSize:16,color:C.textMuted}}>⏳ Chargement...</div></div>;
  return (
    <div style={{maxWidth:430,margin:"0 auto",minHeight:"100vh",background:C.cream,fontFamily:"'Nunito','Segoe UI',sans-serif",position:"relative",display:"flex",flexDirection:"column"}}>
      {screen==="splash"&&<SplashScreen onDone={()=>setScreen(profile?"menu":"login")}/>}
      {screen==="login"&&<LoginScreen onGoRegister={()=>setScreen("register")} onAdminRequest={onAdminRequest}/>}
      {screen==="register"&&<RegisterScreen onGoLogin={()=>setScreen("login")}/>}
      {screen==="menu"&&profile&&<MenuScreen cart={cart} setCart={setCart} onCheckout={()=>setScreen("cart")} onLogout={()=>{signOut();setScreen("login");}}/>}
      {screen==="cart"&&<CartScreen cart={cart} setCart={setCart} onOrder={doOrder} onBack={()=>setScreen("menu")}/>}
      {screen==="tracking"&&<TrackingScreen orderId={currentOrderId} onBack={()=>setScreen("menu")}/>}
    </div>
  );
}

export default function App() {
  const [showAdminLogin,setShowAdminLogin]=useState(false);
  return (
    <StoreProvider>
      <style>{`@import url('https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700;800;900&display=swap');*{box-sizing:border-box;margin:0;padding:0;}body{background:#1A2E0A;}`}</style>
      <AppRouter showAdminLogin={showAdminLogin} setShowAdminLogin={setShowAdminLogin}/>
    </StoreProvider>
  );
}

function AppRouter({ showAdminLogin, setShowAdminLogin }) {
  const { session, isAdmin, loading } = useStore();
  if(loading) return <div style={{display:"flex",alignItems:"center",justifyContent:"center",minHeight:"100vh",background:"#1A2E0A"}}><div style={{color:"#fff",fontSize:16,fontWeight:700}}>⏳ Connexion à U'Break...</div></div>;
  if(isAdmin) return <AdminApp onLogout={()=>{}}/>;
  return (
    <>
      <ClientApp onAdminRequest={()=>setShowAdminLogin(true)}/>
      {showAdminLogin&&<AdminLoginModal onCancel={()=>setShowAdminLogin(false)}/>}
    </>
  );
}
