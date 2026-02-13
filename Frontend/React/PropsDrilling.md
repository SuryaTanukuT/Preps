# Props Drilling (interview perspective)
Props drilling is when you pass props through multiple intermediate components just to deliver data or callbacks to a deeply nested child — even though the middle components don’t use them.
Props drilling is passing data through multiple intermediate components just to reach a deep child, even when those intermediates don’t need it. It mainly hurts maintainability and increases coupling. For app-wide concerns I use Context, for complex shared state I use a store like Redux/Zustand, and sometimes I avoid drilling by composition or moving state closer to where it’s actually used.
Why it happensBecause React’s default data flow is top → down via props.If a deep child needs data from a far parent, you end up forwarding props layer by layer.
# Why it’s a problemBoilerplate: every layer has to accept and forward propsTight coupling: intermediate components become dependent on props they don’t care aboutHard maintenance: renaming/changing a prop touches many filesRefactor pain: moving components around becomes harder
Interview line:“Props drilling isn’t a runtime performance issue; it’s mainly a maintainability and coupling issue.”
# How to avoid it (solutions)1) Context API (most common)Good for global-ish data:theme, auth user, language, permissions
2) State management storeRedux / Zustand / Recoil, etc.Good when:many unrelated components need shared statecomplex updates, caching, derived state
3) Component compositionInstead of passing values, pass children or components:
<Layout sidebar={<Sidebar user={user} />} />This reduces the number of layers that need to know about the prop.
4) Move state closer to where it’s usedIf only a small part needs the data, keep the state in that subtree.

