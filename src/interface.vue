
<script>
  import Inputmask from 'inputmask';
  import { defineComponent, ref, reactive, onUnmounted, nextTick, watch, toRaw } from "vue";

export default defineComponent({
  props: ['value'],
  emits : ['input'],
  setup(props, { emit }) {
    const horaires = [
      {'jour' : 'lundi', horaires : '1000130014002000'},
      {'jour' : 'mardi', horaires : '1000130014002000'},
      {'jour' : 'mercredi', horaires : '1000130014002000'},
      {'jour' : 'jeudi', horaires : '1000130014002000'},
      {'jour' : 'vendredi', horaires : '1000130014002000'},
      {'jour' : 'samedi', horaires : '1000130014002000'},
      {'jour' : 'dimanche', horaires : null }
    ];
    let results = reactive([...(JSON.parse(props.value) || horaires)]);
    let backups = reactive([]);
    const message=reactive([]);
    let closedOnes = ref(horaires.map(c=> !c.horaires || c.horaires === ''));
    const invalids = reactive([false,false,false,false,false,false,false])
    let inputs = reactive([]);
    let init = ref(false);

    const setInput = (el) => {
      if (!el)
        return;
      const index = horaires?.findIndex?.(h => h.jour === el.id);
      if (el && isFinite(index) && !inputs[index]) {
          inputs[index]=new Inputmask('(Hh:Mm - Hh:Mm \\/ ){1,3}|(U)|(S)', { 
          groupSeparator: "\\/ ",
          jitMasking : true,
          clearMaskOnLostFocus: false, 
          clearIncomplete: false,
          autoUnmask : true,
          tabThrough: false,
          nullable: true,
          oncomplete: () => {
            console.log('oncomplete');
            results[index].horaires = inputs[index].unmaskedvalue();
            inputs[index].setValue(results[index].horaires);
            emit('input', JSON.stringify(toRaw(results)));
          },
          onKeyValidation: () => {
            el.style.borderColor = isInvalid(index) ? 'red' : 'green'
            results[index].horaires = inputs[index].unmaskedvalue();
          },
          definitions : {
            'H' : { validator : "[0-2]" },
            'h' : { validator : "[0-9]"},
            'M' : { validator : "[0-5]" },
            'm' : {validator : "[0-9]" },
            'U' : { validator : "O" },
            'S' : { validator : "I" }
          }
        }).mask(el);
      }
    };

    const isInvalid = (index) => {
      let orderedTimes;
      const val = inputs[index].unmaskedvalue()
      const length = val.length;
      const vals = val.match(/.{1,4}/g)
      if (vals && vals.length < 2)
        orderedTimes = true;
      else if (Array.isArray(vals) && vals.every(v => v.length === 4)) { // On découpe tout ce petit monde en 4 caractères

        const pseudoSortedTimes = ([...vals].sort((a,b) => {
          if (parseInt(a) < 400) // On considère que les heures du jour vont jusqu'à 4 du matin
            a = `${2000}+${parseInt(a)}`
          return a-b
        }));

        orderedTimes = pseudoSortedTimes + ' ' === vals + ' '; // On vérifie que les heures sont bien ordonnées en sortant et comparant à la valeur originale

        if (!orderedTimes) {
          message[index] = 'Les horaires doivent être ordonnés';
        } else {
          message[index] = '';
        }
      }
      invalids[index] = !(orderedTimes && inputs[index].isValid() && ( (length % 8) ===0  || (length <= 1)));
      return invalids[index];
    } 

    const handleClosed = (index) => {
      if (!index)
        return;
      closedOnes.value[index] = !closedOnes.value[index];
      if (!closedOnes.value[index]) {
        results[index].horaires = backups[index];
        inputs[index].setValue(backups[index]);
      }
      else {
        backups[index] = inputs[index].unmaskedvalue() || null;
        results[index].horaires = ''
      }
      emit('input', JSON.stringify(toRaw(results)))
    }

    onUnmounted(() => {
      inputs = [];
    });

    const handleHoraires = () => {
      nextTick().then(()=>{
        if (init.value) {
          results = reactive([...(JSON.parse(props.value) || horaires)]);
          backups.length && backups.forEach((b,i)=>results[i].horaires = b);
          inputs.forEach((i,j) => inputs[j].setValue(results[j]?.horaires || ''));
          emit('input', JSON.stringify(toRaw(results)));
        }
        else if (!init.value) {
          backups=results.map(c=> c.horaires || ' ');
          results = [];
          inputs = [];
          emit('input', null);
        }
      });
    }

    watch(
			() => props.value,
			(newVal, oldVal) => {
				if (oldVal === newVal)
          return;
				else if (newVal) {
          init.value = true;
          nextTick().then(()=>{
            results = reactive(JSON.parse(props.value));
            inputs.forEach((i,j) => inputs[j].setValue(results[j]?.horaires || ''));
          });
				}
			}
		);

    return {
      setInput,
      handleClosed,
      results,
      closedOnes,
      message,
      handleHoraires,
      init
    }
  }
});
</script>

<template>
  <div class="container-horaires">
    <div class="tg-list">
      <div class="tg-list-item">
        <h4>Activer Les horaires</h4>
        <input class="tgl tgl-light" id="cb1" v-model="init" type="checkbox" @change="handleHoraires"/>
        <label class="tgl-btn" for="cb1"></label>
      </div>
    </div>

    <ul class="listhoraires" v-if="init" >
      <li class="item" v-for="(horaire, index) in results" >
        <div class="liwrap">
          <label for="horaire">Horaires du {{horaire.jour}}</label>
          <input 
            :disabled="closedOnes[index]"
            :id="horaire.jour"
            :ref="setInput"
            name="horaire"
          />
          <span v-if="message[index]" class="message">{{message[index]}}</span>
        </div>
        <div class="liwrap">
          <label for="closed">Fermé?</label>
          <input name="closed" value="index" v-model="closedOnes[index]" type="checkbox" @click="handleClosed(index)" />
        </div>
      </li>
    </ul>
  </div>
</template>


<style>

.listhoraires {
  display :flex;
  flex-wrap: wrap;
}
.liwrap {
  display :flex;
  flex-direction: column;
  margin-right: 1rem;
  margin-bottom: 1rem;
  width : 100%;
}
.listhoraires li {
  display:flex;
  flex-direction : row;
  width: 100%;
}
.liwrap input:disabled {
  color : #777;
  background : #333;
}
.liwrap input[type="text"] {
  width: 100%;
  border:1px solid;
  border-radius: 4px;
}
.liwrap input[type="checkbox"] {
  appearance: auto;
  width: 2rem;
  height: 2rem;
}
.liwrap .message {
  color : red;
}
.redval-row {
  display :flex;
  flex-direction: row;
  margin-bottom: 1rem;
}
.redval-row > input {
  width: 2rem;
  height: 2rem;
  margin-left: 2rem;
}


.tg-list {
  text-align: center;
  display: flex;
  align-items: center;
}

.tg-list-item {
  margin-left: 1em;
  margin-bottom: 1em;
  display: flex;
  align-items: center;
}
.tg-list-item > h4 {
  margin-right: 1rem;
}
.tgl {
  display: none;
}
.tgl, .tgl:after, .tgl:before, .tgl *, .tgl *:after, .tgl *:before, .tgl + .tgl-btn {
  box-sizing: border-box;
}
.tgl::-moz-selection, .tgl:after::-moz-selection, .tgl:before::-moz-selection, .tgl *::-moz-selection, .tgl *:after::-moz-selection, .tgl *:before::-moz-selection, .tgl + .tgl-btn::-moz-selection {
  background: none;
}
.tgl::selection, .tgl:after::selection, .tgl:before::selection, .tgl *::selection, .tgl *:after::selection, .tgl *:before::selection, .tgl + .tgl-btn::selection {
  background: none;
}
.tgl + .tgl-btn {
  outline: 0;
  display: block;
  width: 2.75em;
  height: 1.5em;
  position: relative;
  cursor: pointer;
  -webkit-user-select: none;
     -moz-user-select: none;
      -ms-user-select: none;
          user-select: none;
}
.tgl + .tgl-btn:after, .tgl + .tgl-btn:before {
  position: relative;
  display: block;
  content: "";
  width: 50%;
  height: 100%;
}
.tgl + .tgl-btn:after {
  left: 0;
}
.tgl + .tgl-btn:before {
  display: none;
}
.tgl:checked + .tgl-btn:after {
  left: 50%;
}

.tgl-light + .tgl-btn {
  background: #ccc;
  border-radius: 2em;
  padding: 2px;
  transition: all 0.4s ease;
}
.tgl-light + .tgl-btn:after {
  border-radius: 50%;
  background: #fff;
  transition: all 0.2s ease;
}
.tgl-light:checked + .tgl-btn {
  background: #FFC23B;
}


</style>

