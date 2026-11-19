# Serialization

Serialization

Author

Benedict Thekkel

Model -\> Serializer -\> View -\> URL -\> Project URL

## Basic config

``` python
class ClientSerializer(serializers.ModelSerializer):
    # practitioners = PractitionerSerializer(many=True, read_only=True)

    class Meta:
        model = Client
        fields = '__all__'
```

## Adding Custom Fields

``` python
class PractitionerSerializer(serializers.ModelSerializer):
    client_ids = serializers.PrimaryKeyRelatedField(many=True, read_only=True, source='clients')

    class Meta:
        model = Practitioner
        fields = '__all__'
```

## Adding Relations Fields

``` python
class QuestionnaireSerializer(serializers.ModelSerializer):
    questions = QuestionSerializer(many=True, read_only=True)

    class Meta:
        model = Questionnaire
        fields = '__all__'
```

Back to top
